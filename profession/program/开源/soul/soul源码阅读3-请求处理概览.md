# Soul 源码阅读（三）请求处理概览
***
## 简介
&ensp;&ensp;&ensp;&ensp;基于上篇：[Soul 源码阅读（二）代码初步运行]()的配置，这次debug下请求处理的大致路径,验证网关模型的路径

### 详细流程记录
&ensp;&ensp;&ensp;&ensp;上篇中我们配置了 Divide 插件，让 http://localhost:9195 ,转发到了后台服务器 http://localhost:8082 上，首先不打断点运行，查看运行日志，找一个切入点：

```shell script
o.d.soul.plugin.base.AbstractSoulPlugin  : divide selector success match , selector name :neety8082
o.d.soul.plugin.base.AbstractSoulPlugin  : divide selector success match , selector name :neety8082
o.d.s.plugin.httpclient.WebClientPlugin  : The request urlPath is http://localhost:8082, retryTimes is 1
```

&ensp;&ensp;&ensp;&ensp;上面的日志中一个比较明显的 divide 相关的日志，是 AbstractSoulPlugin 打印出来的，win下双击shift，搜索 AbstractSoulPlugin 进入，发现是一个接口，IDEA左边向下的箭头查看它的实现，发现有一个熟系的 DividePlugin 实现类，点击进入，在一个明显的 doExecute 函数上打上断点，发起请求：http://localhost:9195

&ensp;&ensp;&ensp;&ensp;通过函数调用栈，发送调用的是 SoulWebHandler ，从下面的函数中可以大致看出这是一个循环遍历，遍历 plugins 进行操作

```java
    public Mono<Void> execute(final ServerWebExchange exchange) {
            return Mono.defer(() -> {
                if (this.index < plugins.size()) {
                    SoulPlugin plugin = plugins.get(this.index++);
                    Boolean skip = plugin.skip(exchange);
                    if (skip) {
                        return this.execute(exchange);
                    }
                    return plugin.execute(exchange, this);
                }
                return Mono.empty();
            });
        }
```

&ensp;&ensp;&ensp;&ensp;再次往前看调用栈，发送 SoulWebHandler 调用了 SoulWebHandler 的 execute

```java
    public Mono<Void> handle(@NonNull final ServerWebExchange exchange) {
        MetricsTrackerFacade.getInstance().counterInc(MetricsLabelEnum.REQUEST_TOTAL.getName());
        Optional<HistogramMetricsTrackerDelegate> startTimer = MetricsTrackerFacade.getInstance().histogramStartTimer(MetricsLabelEnum.REQUEST_LATENCY.getName());
        // new DefaultSoulPluginChain(plugins).execute(exchange) 明显的调用关系
        return new DefaultSoulPluginChain(plugins).execute(exchange).subscribeOn(scheduler)
                .doOnSuccess(t -> startTimer.ifPresent(time -> MetricsTrackerFacade.getInstance().histogramObserveDuration(time)));
    }
```

&ensp;&ensp;&ensp;&ensp;再往前发现看不懂了，没用明显的函数传递关系，我们在上面的函数打上端口，重启程序，再次发送请求

&ensp;&ensp;&ensp;&ensp;断点进来后，查看调用栈，发送调用上面函数的是 DefaultWebFilterChain

```java
    public Mono<Void> filter(ServerWebExchange exchange) {
        return Mono.defer(() -> {
            // 当下面都为null的时候进行调用
            return this.currentFilter != null && this.chain != null ? this.invokeFilter(this.currentFilter, this.chain, exchange) : this.handler.handle(exchange);
        });
    }
```

&ensp;&ensp;&ensp;&ensp;再往前查看，调用栈又看不懂了，再次在上面的函数打上断点，重启，发请求，下面就直接写类和相关函数，有特别的地方就叫点说明

&ensp;&ensp;&ensp;&ensp;来到 FilteringWebHandler

```java
    public Mono<Void> handle(ServerWebExchange exchange) {
        return this.chain.filter(exchange);
    }
```

&ensp;&ensp;&ensp;&ensp;继续来到 WebHandlerDecorator

```java
    public Mono<Void> handle(ServerWebExchange exchange) {
        return this.delegate.handle(exchange);
    }
```

&ensp;&ensp;&ensp;&ensp;来到 ExceptionHandlingWebHandler

```java
public Mono<Void> handle(ServerWebExchange exchange) {
        Mono completion;
        try {
            // 在这进行调用
            completion = super.handle(exchange);
        } catch (Throwable var5) {
            completion = Mono.error(var5);
        }

        WebExceptionHandler handler;
        for(Iterator var3 = this.exceptionHandlers.iterator(); var3.hasNext(); completion = completion.onErrorResume((ex) -> {
            return handler.handle(exchange, ex);
        })) {
            handler = (WebExceptionHandler)var3.next();
        }

        return completion;
    }
```

&ensp;&ensp;&ensp;&ensp;继续来到 HttpWebHandlerAdapter ，这个类有点关键，看到在前面一直传递的变量：exchange，exchange在这个类中生成，传递给后面的函数进行调用，而且是使用response和request生成的

```java
    public Mono<Void> handle(ServerHttpRequest request, ServerHttpResponse response) {
        if (this.forwardedHeaderTransformer != null) {
            request = this.forwardedHeaderTransformer.apply(request);
        }

        // 重点变量 exchange 的生成
        ServerWebExchange exchange = this.createExchange(request, response);
        LogFormatUtils.traceDebug(logger, (traceOn) -> {
            return exchange.getLogPrefix() + this.formatRequest(exchange.getRequest()) + (traceOn ? ", headers=" + this.formatHeaders(exchange.getRequest().getHeaders()) : "");
        });
        // this.getDelegate().handle(exchange)
        // 通过debug可以看到 getDelete 得到的是 ExceptionHandlingWebHandler，那调用就是在这
        Mono var10000 = this.getDelegate().handle(exchange).doOnSuccess((aVoid) -> {
            this.logResponse(exchange);
        }).onErrorResume((ex) -> {
            return this.handleUnresolvedError(exchange, ex);
        });
        response.getClass();
        return var10000.then(Mono.defer(response::setComplete));
    }
```

&ensp;&ensp;&ensp;&ensp;继续走到 ReactiveWebServerApplicationContext

```java
    public Mono<Void> handle(ServerHttpRequest request, ServerHttpResponse response) {
            return this.handler.handle(request, response);
        }
```

&ensp;&ensp;&ensp;&ensp;继续走到 ReactorHttpHandlerAdapter

```java
    public Mono<Void> apply(HttpServerRequest reactorRequest, HttpServerResponse reactorResponse) {
        NettyDataBufferFactory bufferFactory = new NettyDataBufferFactory(reactorResponse.alloc());

        try {
            // exchange 需要的 request 和 response 的生成
            ReactorServerHttpRequest request = new ReactorServerHttpRequest(reactorRequest, bufferFactory);
            ServerHttpResponse response = new ReactorServerHttpResponse(reactorResponse, bufferFactory);
            if (request.getMethod() == HttpMethod.HEAD) {
                response = new HttpHeadResponseDecorator((ServerHttpResponse)response);
            }

            return this.httpHandler.handle(request, (ServerHttpResponse)response).doOnError((ex) -> {
                logger.trace(request.getLogPrefix() + "Failed to complete: " + ex.getMessage());
            }).doOnSuccess((aVoid) -> {
                logger.trace(request.getLogPrefix() + "Handling completed");
            });
        } catch (URISyntaxException var6) {
            if (logger.isDebugEnabled()) {
                logger.debug("Failed to get request URI: " + var6.getMessage());
            }

            reactorResponse.status(HttpResponseStatus.BAD_REQUEST);
            return Mono.empty();
        }
    }
```

&ensp;&ensp;&ensp;&ensp;继续走到 HttpServerHandle

```java
    public void onStateChange(Connection connection, State newState) {
        if (newState == HttpServerState.REQUEST_RECEIVED) {
            try {
                if (log.isDebugEnabled()) {
                    log.debug(ReactorNetty.format(connection.channel(), "Handler is being applied: {}"), new Object[]{this.handler});
                }

                HttpServerOperations ops = (HttpServerOperations)connection;
                Mono.fromDirect((Publisher)this.handler.apply(ops, ops)).subscribe(ops.disposeSubscriber());
            } catch (Throwable var4) {
                log.error(ReactorNetty.format(connection.channel(), ""), var4);
                connection.channel().close();
            }
        }

    }
```

&ensp;&ensp;&ensp;&ensp;继续走到 TcpServerBind

```java
        public void onStateChange(Connection connection, State newState) {
            if (newState == State.DISCONNECTING && connection.channel().isActive() && !connection.isPersistent()) {
                connection.dispose();
            }

            this.childObs.onStateChange(connection, newState);
        }
```

&ensp;&ensp;&ensp;&ensp;走到了很关键的一个： HttpServerOperations ，下面这个函数的 ctx 和 msg 也太熟悉不过了，明显的 netty 的 handler处理，请求入口

```java
    protected void onInboundNext(ChannelHandlerContext ctx, Object msg) {
        if (msg instanceof HttpRequest) {
            try {
                // 调用
                this.listener().onStateChange(this, HttpServerState.REQUEST_RECEIVED);
            } catch (Exception var4) {
                this.onInboundError(var4);
                ReferenceCountUtil.release(msg);
                return;
            }

            if (msg instanceof FullHttpRequest) {
                super.onInboundNext(ctx, msg);
                if (this.isHttp2()) {
                    this.onInboundComplete();
                }
            }

        } else {
            if (msg instanceof HttpContent) {
                if (msg != LastHttpContent.EMPTY_LAST_CONTENT) {
                    super.onInboundNext(ctx, msg);
                }

                if (msg instanceof LastHttpContent) {
                    this.onInboundComplete();
                }
            } else {
                super.onInboundNext(ctx, msg);
            }

        }
    }
```

&ensp;&ensp;&ensp;&ensp;这个时候走到头了，我们跳出来看一看，梳理一下目前所得，发现我们搞清楚了一个请求接受，然后到 divide 的处理过程，梳理下大致如下：

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
- SoulWebHandler ：plugins调用链
- DividePlugin ：plugin具体处理

&ensp;&ensp;&ensp;&ensp;这个时候参考网关模型，发现路由匹配之类的没有看到，没办法，细节部分没有清楚的就是: SoulWebHandler ,它的plugin调用的部分没有细看，于是我们进行如下的函数的debug，进入各个函数的调用（进入subscribe之类的时候就跳出来，点击进入下一个断点，IDEA debug左上角的箭头）

```java
        public Mono<Void> execute(final ServerWebExchange exchange) {
            return Mono.defer(() -> {
                if (this.index < plugins.size()) {
                    SoulPlugin plugin = plugins.get(this.index++);
                    Boolean skip = plugin.skip(exchange);
                    if (skip) {
                        return this.execute(exchange);
                    }
                    return plugin.execute(exchange, this);
                }
                return Mono.empty();
            });
        }
```

&ensp;&ensp;&ensp;&ensp;我们逐步调试上面的那个函数，查看变量： plugins，内容大致如下，后面false和true是变量 skip。发现是true就不执行，看函数也能大致猜的到

GlobalPlugin : false
SignPlugin : false
WafPlugin: false
RateLimiterPlugin : false
HystrixPlugin : false
Resilience4JPlugin : false
DividePlugin : false
WebClientPulugin : false
WebsocketPlugin : true
BodyParamPlugin : false
AlibabaDubblePlugin : true
MonitorPlugin : false
WebClientResponsePlugin : false
DubboResponsePlugin : true

&ensp;&ensp;&ensp;&ensp;调试的时候跟着进去，进去以后一步一步的走即可

&ensp;&ensp;&ensp;&ensp;我们调试进入前几个plugin都是没有执行到下面代码中的if，在 divide plugin 执行了，我们跟着进入看看，看到了疑似路由匹配的 rules，还有 match，猜测这是路由匹配相关

```java
    public Mono<Void> execute(final ServerWebExchange exchange, final SoulPluginChain chain) {
        String pluginName = named();
        final PluginData pluginData = BaseDataCache.getInstance().obtainPluginData(pluginName);
        if (pluginData != null && pluginData.getEnabled()) {
            final Collection<SelectorData> selectors = BaseDataCache.getInstance().obtainSelectorData(pluginName);
            if (CollectionUtils.isEmpty(selectors)) {
                return handleSelectorIsNull(pluginName, exchange, chain);
            }
            final SelectorData selectorData = matchSelector(exchange, selectors);
            if (Objects.isNull(selectorData)) {
                return handleSelectorIsNull(pluginName, exchange, chain);
            }
            selectorLog(selectorData, pluginName);
            final List<RuleData> rules = BaseDataCache.getInstance().obtainRuleData(selectorData.getId());
            if (CollectionUtils.isEmpty(rules)) {
                return handleRuleIsNull(pluginName, exchange, chain);
            }
            RuleData rule;
            if (selectorData.getType() == SelectorTypeEnum.FULL_FLOW.getCode()) {
                //get last
                rule = rules.get(rules.size() - 1);
            } else {
                // divide plugin 执行到这步，在rules，发现我们配置的规则，猜测这里是路由匹配
                rule = matchRule(exchange, rules);
            }
            if (Objects.isNull(rule)) {
                return handleRuleIsNull(pluginName, exchange, chain);
            }
            ruleLog(rule, pluginName);
            return doExecute(exchange, chain, selectorData, rule);
        }
        return chain.execute(exchange);
    }
```

&ensp;&ensp;&ensp;&ensp;继续debug，进入： WebClientPlugin ，看到了疑似发送请求给后台服务器，相关代码如下：

```java
    public Mono<Void> execute(final ServerWebExchange exchange, final SoulPluginChain chain) {
        final SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
        assert soulContext != null;
        String urlPath = exchange.getAttribute(Constants.HTTP_URL);
        if (StringUtils.isEmpty(urlPath)) {
            Object error = SoulResultWrap.error(SoulResultEnum.CANNOT_FIND_URL.getCode(), SoulResultEnum.CANNOT_FIND_URL.getMsg(), null);
            return WebFluxResultUtils.result(exchange, error);
        }
        long timeout = (long) Optional.ofNullable(exchange.getAttribute(Constants.HTTP_TIME_OUT)).orElse(3000L);
        int retryTimes = (int) Optional.ofNullable(exchange.getAttribute(Constants.HTTP_RETRY)).orElse(0);
        log.info("The request urlPath is {}, retryTimes is {}", urlPath, retryTimes);
        HttpMethod method = HttpMethod.valueOf(exchange.getRequest().getMethodValue());
        WebClient.RequestBodySpec requestBodySpec = webClient.method(method).uri(urlPath);
        return handleRequestBody(requestBodySpec, exchange, timeout, retryTimes, chain);
    }

    private Mono<Void> handleRequestBody(final WebClient.RequestBodySpec requestBodySpec,
                                         final ServerWebExchange exchange,
                                         final long timeout,
                                         final int retryTimes,
                                         final SoulPluginChain chain) {
        // 下面这段代码太想前端的 ajax 调用了，猜测这是 请求发送
        return requestBodySpec.headers(httpHeaders -> {
            httpHeaders.addAll(exchange.getRequest().getHeaders());
            httpHeaders.remove(HttpHeaders.HOST);
        })
                .contentType(buildMediaType(exchange))
                .body(BodyInserters.fromDataBuffers(exchange.getRequest().getBody()))
                .exchange()
                .doOnError(e -> log.error(e.getMessage()))
                .timeout(Duration.ofMillis(timeout))
                .retryWhen(Retry.onlyIf(x -> x.exception() instanceof ConnectTimeoutException)
                    .retryMax(retryTimes)
                    .backoff(Backoff.exponential(Duration.ofMillis(200), Duration.ofSeconds(20), 2, true)))
                .flatMap(e -> doNext(e, exchange, chain));

    }

    private Mono<Void> doNext(final ClientResponse res, final ServerWebExchange exchange, final SoulPluginChain chain) {
        if (res.statusCode().is2xxSuccessful()) {
            exchange.getAttributes().put(Constants.CLIENT_RESPONSE_RESULT_TYPE, ResultEnum.SUCCESS.getName());
        } else {
            exchange.getAttributes().put(Constants.CLIENT_RESPONSE_RESULT_TYPE, ResultEnum.ERROR.getName());
        }
        exchange.getAttributes().put(Constants.CLIENT_RESPONSE_ATTR, res);
        return chain.execute(exchange);
    }
```

&ensp;&ensp;&ensp;&ensp;继续进入，来到： WebClientResponsePlugin ，发现疑似 response 返回给客户端的代码

```java
    public Mono<Void> execute(final ServerWebExchange exchange, final SoulPluginChain chain) {
        return chain.execute(exchange).then(Mono.defer(() -> {
            ServerHttpResponse response = exchange.getResponse();
            ClientResponse clientResponse = exchange.getAttribute(Constants.CLIENT_RESPONSE_ATTR);
            if (Objects.isNull(clientResponse)
                    || response.getStatusCode() == HttpStatus.BAD_GATEWAY
                    || response.getStatusCode() == HttpStatus.INTERNAL_SERVER_ERROR) {
                Object error = SoulResultWrap.error(SoulResultEnum.SERVICE_RESULT_ERROR.getCode(), SoulResultEnum.SERVICE_RESULT_ERROR.getMsg(), null);
                return WebFluxResultUtils.result(exchange, error);
            }
            if (response.getStatusCode() == HttpStatus.GATEWAY_TIMEOUT) {
                Object error = SoulResultWrap.error(SoulResultEnum.SERVICE_TIMEOUT.getCode(), SoulResultEnum.SERVICE_TIMEOUT.getMsg(), null);
                return WebFluxResultUtils.result(exchange, error);
            }
            response.setStatusCode(clientResponse.statusCode());
            response.getCookies().putAll(clientResponse.cookies());
            response.getHeaders().putAll(clientResponse.headers().asHttpHeaders());
            // 疑似响应返回
            return response.writeWith(clientResponse.body(BodyExtractors.toDataBuffers()));
        }));
    }
```

&ensp;&ensp;&ensp;&ensp;经过一通调试，发现好像Soul 是有可能 filter 相关的操作在前面，但好像 filter 在前，感觉也可以

&ensp;&ensp;&ensp;&ensp;今天太困了，明天继续验证今天调试时的想法