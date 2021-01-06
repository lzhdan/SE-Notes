# Spring Cloud Gateway (八) 定制化filter的配置加载
***
### 定制化filter的来源
&ensp;&ensp;&ensp;&ensp;非 global filter 从 routeLocator 中获取的，跟踪 routeLocator 看其的来源

```java
    protected Mono<Route> lookupRoute(ServerWebExchange exchange) {
        // routeLocator 跟踪堆栈发现是 CachingRouteLocator
		return this.routeLocator.getRoutes()
				// individually filter routes so that filterWhen error delaying is not a
				// problem
				.concatMap(route -> Mono.just(route).filterWhen(r -> {
					// add the current route we are testing
					exchange.getAttributes().put(GATEWAY_PREDICATE_ROUTE_ATTR, r.getId());
					return r.getPredicate().apply(exchange);
				})
						// instead of immediately stopping main flux due to error, log and
						// swallow it
						.doOnError(e -> logger.error(
								"Error applying predicate for route: " + route.getId(),
								e))
						.onErrorResume(e -> Mono.empty()))
				// .defaultIfEmpty() put a static Route not found
				// or .switchIfEmpty()
				// .switchIfEmpty(Mono.<Route>empty().log("noroute"))
				.next()
				// TODO: error handling
				.map(route -> {
					if (logger.isDebugEnabled()) {
						logger.debug("Route matched: " + route.getId());
					}
					validateRoute(route, exchange);
					return route;
				});

		/*
		 * TODO: trace logging if (logger.isTraceEnabled()) {
		 * logger.trace("RouteDefinition did not match: " + routeDefinition.getId()); }
		 */
	}
```

&ensp;&ensp;&ensp;&ensp;再跟踪 CachingRouteLocator ，发现是从 CompositeRouteLocator 来的

```java
    public CachingRouteLocator(RouteLocator delegate) {
		this.delegate = delegate;

        // 这里提供一种Flux流缓存构造方式，他的意思是，当cache没有的时候，
        // 我们执行this.delegate.getRoutes().sort(AnnotationAwareOrderComparator.INSTANCE)获得Flux流，并把这个结果写入cache这个Map。
        // 这不会马上执行，当有subscribe才会执行。是懒加载方式。
		routes = CacheFlux.lookup(cache, CACHE_KEY, Route.class)
				.onCacheMissResume(this::fetch);
	}

    private Flux<Route> fetch() {
        // 这个方法是一个核心的方法。负责把所有routeDefinition转换为route。是连接RouteDefinitionLocator和RouteLocator的通道
		return this.delegate.getRoutes().sort(AnnotationAwareOrderComparator.INSTANCE);
	}


    @Bean
	@Primary
	@ConditionalOnMissingBean(name = "cachedCompositeRouteLocator")
	// TODO: property to disable composite?
	public RouteLocator cachedCompositeRouteLocator(List<RouteLocator> routeLocators) {
        // routeLocators 里面有大量的 定制化的 filter factory
		return new CachingRouteLocator(new CompositeRouteLocator(Flux.fromIterable(routeLocators)));
	}

    public Flux<Route> getRoutes() {
		return this.delegates.flatMapSequential(RouteLocator::getRoutes);
	}
```

&ensp;&ensp;&ensp;&ensp;跟踪 CompositeRouteLocator 发现是初始化加载就初始化好了，大致流程是知道，细节后面再看，因为这部分属于加载了，后面分析这部分的时候再分析