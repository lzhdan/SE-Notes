# Soul网关源码阅读（六）请求类型探索
***
## 简介
&ensp;&ensp;&ensp;&ensp;在上几篇文章中分析了请求的处理流程，HTTP和RPC请求处理是互斥的，通过请求类型来判断，这篇文章来探索下请求类型的前世今生

## 源码分析
&ensp;&ensp;&ensp;&ensp;通过前面的分析，通过请求类型判断是否进入这个plugin进行执行，大致如下：

```java
    public Boolean skip(final ServerWebExchange exchange) {
        final SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
        return !Objects.equals(Objects.requireNonNull(soulContext).getRpcType(), RpcTypeEnum.HTTP.getName());
    }
```

### Plugin Skip 函数
&ensp;&ensp;&ensp;&ensp;我们探索下那些plugin有实现skip函数

&ensp;&ensp;&ensp;&ensp;DividePlugin,判断是不是HTTP

```java
    public Boolean skip(final ServerWebExchange exchange) {
        final SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
        return !Objects.equals(Objects.requireNonNull(soulContext).getRpcType(), RpcTypeEnum.HTTP.getName());
    }`
```

&ensp;&ensp;&ensp;&ensp;WebClientPlugin,是HTTP或者SPRING_CLOUD

```java
    public Boolean skip(final ServerWebExchange exchange) {
        final SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
        assert soulContext != null;
        return !Objects.equals(RpcTypeEnum.HTTP.getName(), soulContext.getRpcType())
                && !Objects.equals(RpcTypeEnum.SPRING_CLOUD.getName(), soulContext.getRpcType());
    }
```


&ensp;&ensp;&ensp;&ensp;WebsocketPlugin,是WEB_SOCKET

```java
    public Boolean skip(final ServerWebExchange exchange) {
        final SoulContext body = exchange.getAttribute(Constants.CONTEXT);
        return !Objects.equals(Objects.requireNonNull(body).getRpcType(), RpcTypeEnum.WEB_SOCKET.getName());
    }
```

&ensp;&ensp;&ensp;&ensp;AlibabaDubboPlugin,类型是DUBBO

```java
    public Boolean skip(final ServerWebExchange exchange) {
        final SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
        assert soulContext != null;
        return !Objects.equals(soulContext.getRpcType(), RpcTypeEnum.DUBBO.getName());
    }
```

&ensp;&ensp;&ensp;&ensp;WebClientResponsePlugin，类型是HTTP或者SPRING_CLOUD

```java
    public Boolean skip(final ServerWebExchange exchange) {
        final SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
        assert soulContext != null;
        return !Objects.equals(RpcTypeEnum.HTTP.getName(), soulContext.getRpcType())
                && !Objects.equals(RpcTypeEnum.SPRING_CLOUD.getName(), soulContext.getRpcType());
    }
```

&ensp;&ensp;&ensp;&ensp;DubboResponsePlugin，类型是DUBBO

```java
    public Boolean skip(final ServerWebExchange exchange) {
        final SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
        assert soulContext != null;
        return !Objects.equals(soulContext.getRpcType(), RpcTypeEnum.DUBBO.getName());
    }
```

&ensp;&ensp;&ensp;&ensp;两个类型看着有点怪，有时候转不过弯了，需要注意

### 传递链追踪
&ensp;&ensp;&ensp;&ensp;通过前几篇，我们可以总结得到下面的处理链路经过的类

- HttpServerOperations : 明显的netty的请求接收的地方，请求入口
- TcpServerBind
- HttpServerHandle
- ReactorHttpHandlerAdapter ：生成response和request
- ReactiveWebServerApplicationContext
- HttpWebHandlerAdapter ：exchange 的生成
- ExceptionHandlingWebHandler
- WebHandlerDecorator
- FilteringWebHandler
- DefaultWebFilterChain
  - MetricsWebFilter
  - HealthFilter
  - FileSizeFilter
  - WebSocketParamFilter
  - HiddenHttpMethodFilter
- SoulWebHandler ：plugins调用链，后面带type的表明是需要去类型处理判断的
  - GlobalPlugin
  - SignPlugin
  - WafPlugin
  - RateLimiterPlugin
  - HystrixPlugin
  - Resilience4JPlugin
  - DividePlugin : type
  - WebClientPlugin : type
  - WebsocketPlugin : type
  - BodyParamPlugin
  - AlibabaDubblePlugin : type
  - MonitorPlugin
  - WebClientResponsePlugin : type
  - DubboResponsePlugin : type

&ensp;&ensp;&ensp;&ensp;通过上面的链路，可以猜测在DividePlugin之前的类都是有嫌疑进行处理的

&ensp;&ensp;&ensp;&ensp;一个一个类的太多了，看不过来，我们看下取类似相关的代码，大致如下：

```java
final SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
soulContext.getRpcType()
```

&ensp;&ensp;&ensp;&ensp;看看将响应放到exchange中的代码，可以猜测到把Constants.CONTEXT放到exchange的大致代码：

```java
exchange.getAttributes().put(Constants.CLIENT_RESPONSE_RESULT_TYPE, ResultEnum.SUCCESS.getName());

// 可以得到放Constants.CONTEXT的大致代码如下：
exchange.getAttributes().put(Constants.CONTEXT,
```

&ensp;&ensp;&ensp;&ensp;在IDEA中使用ctrl+shift+R，全局搜索，发现在GlobalPlugin中有嫌疑代码：

```java
    public Mono<Void> execute(final ServerWebExchange exchange, final SoulPluginChain chain) {
        final ServerHttpRequest request = exchange.getRequest();
        final HttpHeaders headers = request.getHeaders();
        final String upgrade = headers.getFirst("Upgrade");
        SoulContext soulContext;
        if (StringUtils.isBlank(upgrade) || !"websocket".equals(upgrade)) {
            // 对SoulContext进行操作
            soulContext = builder.build(exchange);
        } else {
            final MultiValueMap<String, String> queryParams = request.getQueryParams();
            // 对SoulContext进行操作
            soulContext = transformMap(queryParams);
        }
        // 更新SoulContext
        exchange.getAttributes().put(Constants.CONTEXT, soulContext);
        return chain.execute(exchange);
    }
```

&ensp;&ensp;&ensp;&ensp;有两处嫌疑：build和transformMap，我们在看看build函数具体细节

```java
    # build(exchange)
    
    public SoulContext build(final ServerWebExchange exchange) {
        final ServerHttpRequest request = exchange.getRequest();
        String path = request.getURI().getPath();
        // 根据path得到一些元数据
        MetaData metaData = MetaDataCache.getInstance().obtain(path);
        if (Objects.nonNull(metaData) && metaData.getEnabled()) {
            exchange.getAttributes().put(Constants.META_DATA, metaData);
        }
        // 再看下去
        return transform(request, metaData);
    }

    private SoulContext transform(final ServerHttpRequest request, final MetaData metaData) {
        final String appKey = request.getHeaders().getFirst(Constants.APP_KEY);
        final String sign = request.getHeaders().getFirst(Constants.SIGN);
        final String timestamp = request.getHeaders().getFirst(Constants.TIMESTAMP);
        SoulContext soulContext = new SoulContext();
        String path = request.getURI().getPath();
        soulContext.setPath(path);
        // 下面都是判断，而且是基于metaData的判断，猜测是在路由注册的时候就携带了相关的类型信息，而且默认是HTTP
        if (Objects.nonNull(metaData) && metaData.getEnabled()) {
            if (RpcTypeEnum.SPRING_CLOUD.getName().equals(metaData.getRpcType())) {
                setSoulContextByHttp(soulContext, path);
                soulContext.setRpcType(metaData.getRpcType());
            } else if (RpcTypeEnum.DUBBO.getName().equals(metaData.getRpcType())) {
                setSoulContextByDubbo(soulContext, metaData);
            } else if (RpcTypeEnum.SOFA.getName().equals(metaData.getRpcType())) {
                setSoulContextBySofa(soulContext, metaData);
            } else if (RpcTypeEnum.TARS.getName().equals(metaData.getRpcType())) {
                setSoulContextByTars(soulContext, metaData);
            } else {
                setSoulContextByHttp(soulContext, path);
                soulContext.setRpcType(RpcTypeEnum.HTTP.getName());
            }
        } else {
            setSoulContextByHttp(soulContext, path);
            soulContext.setRpcType(RpcTypeEnum.HTTP.getName());
        }
        soulContext.setAppKey(appKey);
        soulContext.setSign(sign);
        soulContext.setTimestamp(timestamp);
        soulContext.setStartDateTime(LocalDateTime.now());
        Optional.ofNullable(request.getMethod()).ifPresent(httpMethod -> soulContext.setHttpMethod(httpMethod.name()));
        return soulContext;
    }
```

&ensp;&ensp;&ensp;&ensp;看下另外一个transformMap

```java
    private SoulContext transformMap(final MultiValueMap<String, String> queryParams) {
        SoulContext soulContext = new SoulContext();
        soulContext.setModule(queryParams.getFirst(Constants.MODULE));
        soulContext.setMethod(queryParams.getFirst(Constants.METHOD));
        soulContext.setRpcType(queryParams.getFirst(Constants.RPC_TYPE));
        return soulContext;
    }
```

&ensp;&ensp;&ensp;&ensp;再跟下去感觉要到路由初始化配置了，但有点累了，明天继续再看吧