# Spring Cloud Gateway （三） Filter 链
***
## 简介
&ensp;&ensp;&ensp;&ensp;在上篇中分析了请求在Filter中的传播过程，但不同的请求，应该是有不同的各自的定制的filter，今天来看看如何对请求定制filter链

## 分析过程
&ensp;&ensp;&ensp;&ensp;我们找到上次filter链的循环部分，里面对filter链进行了循环遍历，逐个对每个filter进行了调用，在filter里面对全局的exchange进行了修改，exchange承担了整个流程中的数据。

&ensp;&ensp;&ensp;&ensp;在 FilteringWebHandler 中找到了如何构造特定请求的定制filter链的代码。在下面的代码中，特定请求的filter存放到了exchange中，和全局的filter放到一起排序，形成了上次的filter链路

```java
public class FilteringWebHandler implements WebHandler {
	@Override
	public Mono<Void> handle(ServerWebExchange exchange) {
		// 这里是获取特定转发的filter列表
		Route route = exchange.getRequiredAttribute(GATEWAY_ROUTE_ATTR);
		List<GatewayFilter> gatewayFilters = route.getFilters();

		// 获取全局filter列表，和上面的排序组合成新的filter
		List<GatewayFilter> combined = new ArrayList<>(this.globalFilters);
		combined.addAll(gatewayFilters);
		// TODO: needed or cached?
		AnnotationAwareOrderComparator.sort(combined);

		if (logger.isDebugEnabled()) {
			logger.debug("Sorted gatewayFilterFactories: " + combined);
		}

		// 触发调用
		return new DefaultGatewayFilterChain(combined).filter(exchange);
	}
}
```

&ensp;&ensp;&ensp;&ensp;想查询定位到exchange的来源，但一直没有定位到