# Soul 网关源码阅读（五）Sofa请求处理概览
***
## 简介
&ensp;&ensp;&ensp;&ensp;今天来探索一下Sofa请求处理流程，看看和前面的HTTP、Dubbo有什么异同

## 

soul-examples --> soul-examples-sofa
admin

o.d.s.w.configuration.SoulConfiguration  : load plugin:[global] [org.dromara.soul.plugin.global.GlobalPlugin]
o.d.s.w.configuration.SoulConfiguration  : load plugin:[sign] [org.dromara.soul.plugin.sign.SignPlugin]
o.d.s.w.configuration.SoulConfiguration  : load plugin:[waf] [org.dromara.soul.plugin.waf.WafPlugin]
o.d.s.w.configuration.SoulConfiguration  : load plugin:[rate_limiter] [org.dromara.soul.plugin.ratelimiter.RateLimiterPlugin]
o.d.s.w.configuration.SoulConfiguration  : load plugin:[hystrix] [org.dromara.soul.plugin.hystrix.HystrixPlugin]
o.d.s.w.configuration.SoulConfiguration  : load plugin:[resilience4j] [org.dromara.soul.plugin.resilience4j.Resilience4JPlugin]
o.d.s.w.configuration.SoulConfiguration  : load plugin:[divide] [org.dromara.soul.plugin.divide.DividePlugin]
o.d.s.w.configuration.SoulConfiguration  : load plugin:[webClient] [org.dromara.soul.plugin.httpclient.WebClientPlugin]
o.d.s.w.configuration.SoulConfiguration  : load plugin:[divide] [org.dromara.soul.plugin.divide.websocket.WebSocketPlugin]
o.d.s.w.configuration.SoulConfiguration  : load plugin:[alibaba-dubbo-body-param] [org.dromara.soul.plugin.alibaba.dubbo.param.BodyParamPlugin]
o.d.s.w.configuration.SoulConfiguration  : load plugin:[dubbo] [org.dromara.soul.plugin.alibaba.dubbo.AlibabaDubboPlugin]
o.d.s.w.configuration.SoulConfiguration  : load plugin:[monitor] [org.dromara.soul.plugin.monitor.MonitorPlugin]
o.d.s.w.configuration.SoulConfiguration  : load plugin:[response] [org.dromara.soul.plugin.httpclient.response.WebClientResponsePlugin]
o.d.s.w.configuration.SoulConfiguration  : load plugin:[response] [org.dromara.soul.plugin.alibaba.dubbo.response.DubboResponsePlugin]

```java
    public SoulWebHandler soulWebHandler(final ObjectProvider<List<SoulPlugin>> plugins) {
        List<SoulPlugin> pluginList = plugins.getIfAvailable(Collections::emptyList);
        final List<SoulPlugin> soulPlugins = pluginList.stream()
                .sorted(Comparator.comparingInt(SoulPlugin::getOrder)).collect(Collectors.toList());
        soulPlugins.forEach(soulPlugin -> log.info("load plugin:[{}] [{}]", soulPlugin.named(), soulPlugin.getClass().getName()));
        return new SoulWebHandler(soulPlugins);
    }
```