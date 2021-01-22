# Soul网关源码阅读（十二）数据同步初探
***
## 简介
&ensp;&ensp;&ensp;&ensp;此篇开始进入分析Soul网关的数据同步相关源码，首先看下数据是哪些数据，同步大致原理和方式

## 源码Debug
&ensp;&ensp;&ensp;&ensp;数据这里应该路由相关的数据，在程序中用来进行规则匹配、获取后端服务器真实地址之类的

&ensp;&ensp;&ensp;&ensp;在前面的处理流程的分析中，我们知道进行路由匹配的核心代码如下：

```java
public abstract class AbstractSoulPlugin implements SoulPlugin {
    ......
    public Mono<Void> execute(final ServerWebExchange exchange, final SoulPluginChain chain) {
        String pluginName = named();
        // 获取插件配置
        final PluginData pluginData = BaseDataCache.getInstance().obtainPluginData(pluginName);
        if (pluginData != null && pluginData.getEnabled()) {
            // 获取选择器
            final Collection<SelectorData> selectors = BaseDataCache.getInstance().obtainSelectorData(pluginName);
            .......
            // 获取规则
            final SelectorData selectorData = matchSelector(exchange, selectors);
            .......
        }
        return chain.execute(exchange);
    }
    ......
}
```

&ensp;&ensp;&ensp;&ensp;从上面可以看出数据大致有三种：插件配置数据、选择器数据、规则数据，这些数据都是从 BaseDataCache中获取的，我们来看看其核心代码：

```java
public final class BaseDataCache {
    
    private static final BaseDataCache INSTANCE = new BaseDataCache();
    
    /**
     * pluginName -> PluginData.
     */
    private static final ConcurrentMap<String, PluginData> PLUGIN_MAP = Maps.newConcurrentMap();
    
    /**
     * pluginName -> SelectorData.
     */
    private static final ConcurrentMap<String, List<SelectorData>> SELECTOR_MAP = Maps.newConcurrentMap();
    
    /**
     * selectorId -> RuleData.
     */
    private static final ConcurrentMap<String, List<RuleData>> RULE_MAP = Maps.newConcurrentMap();

    public void cachePluginData(final PluginData pluginData) {
        Optional.ofNullable(pluginData).ifPresent(data -> PLUGIN_MAP.put(data.getName(), data));
    }

    public void removePluginData(final PluginData pluginData) {
        Optional.ofNullable(pluginData).ifPresent(data -> PLUGIN_MAP.remove(data.getName()));
    }
    
    public PluginData obtainPluginData(final String pluginName) {
        return PLUGIN_MAP.get(pluginName);
    }

    public void cacheSelectData(final SelectorData selectorData) {
        Optional.ofNullable(selectorData).ifPresent(this::selectorAccept);
    }

    public void removeSelectData(final SelectorData selectorData) {
        Optional.ofNullable(selectorData).ifPresent(data -> {
            final List<SelectorData> selectorDataList = SELECTOR_MAP.get(data.getPluginName());
            Optional.ofNullable(selectorDataList).ifPresent(list -> list.removeIf(e -> e.getId().equals(data.getId())));
        });
    }
    
    public List<SelectorData> obtainSelectorData(final String pluginName) {
        return SELECTOR_MAP.get(pluginName);
    }

    public void cacheRuleData(final RuleData ruleData) {
        Optional.ofNullable(ruleData).ifPresent(this::ruleAccept);
    }
    
    public void removeRuleData(final RuleData ruleData) {
        Optional.ofNullable(ruleData).ifPresent(data -> {
            final List<RuleData> ruleDataList = RULE_MAP.get(data.getSelectorId());
            Optional.ofNullable(ruleDataList).ifPresent(list -> list.removeIf(rule -> rule.getId().equals(data.getId())));
        });
    }

    public List<RuleData> obtainRuleData(final String selectorId) {
        return RULE_MAP.get(selectorId);
    }
}
```

&ensp;&ensp;&ensp;&ensp;从上面的代码可以看出，这是一个静态单例类，从各个方法的名字大致可以看出基本都是对插件配置数据、选择器配置数据、规则配置数据进行的增删改查

&ensp;&ensp;&ensp;&ensp;其中明显的可以看出 cachexxxx , removexxxx 就是对应的更新（增加和修改对map来说一样的操作）和删除操作

&ensp;&ensp;&ensp;&ensp;下面我们来看下， cachexxx、removexxxxx 的方法被哪些地方进行了引用

&ensp;&ensp;&ensp;&ensp;我们使用IDEA，Alt+F7 查看用那些地方进行了使用，通篇查找下来，处理 BaseDataCache类本身和一些测试相关的类，只有CommonPluginDataSubscriber这个类进行了调用，那这个类就是数据同步的核心类了，大致代码如下：

```java
public class CommonPluginDataSubscriber implements PluginDataSubscriber {
    .......

    // 下面这类函数又进行了一次封装，我们可以看到这里有那么一点小瑕疵，命名有点不统一；onSubscribe这个函数取的不是太好，不能见名识意，有点宽泛了
    @Override
    public void onSubscribe(final PluginData pluginData) {
        subscribeDataHandler(pluginData, DataEventTypeEnum.UPDATE);
    }
    
    @Override
    public void unSubscribe(final PluginData pluginData) {
        subscribeDataHandler(pluginData, DataEventTypeEnum.DELETE);
    }

    public void onSelectorSubscribe(final SelectorData selectorData) {
        subscribeDataHandler(selectorData, DataEventTypeEnum.UPDATE);
    }
    
    @Override
    public void unSelectorSubscribe(final SelectorData selectorData) {
        subscribeDataHandler(selectorData, DataEventTypeEnum.DELETE);
    }

    @Override
    public void onRuleSubscribe(final RuleData ruleData) {
        subscribeDataHandler(ruleData, DataEventTypeEnum.UPDATE);
    }
    
    @Override
    public void unRuleSubscribe(final RuleData ruleData) {
        subscribeDataHandler(ruleData, DataEventTypeEnum.DELETE);
    }
    
    private <T> void subscribeDataHandler(final T classData, final DataEventTypeEnum dataType) {
        Optional.ofNullable(classData).ifPresent(data -> {
            if (data instanceof PluginData) {
                PluginData pluginData = (PluginData) data;
                if (dataType == DataEventTypeEnum.UPDATE) {
                    // 插件配置数据的更新
                    BaseDataCache.getInstance().cachePluginData(pluginData);
                    Optional.ofNullable(handlerMap.get(pluginData.getName())).ifPresent(handler -> handler.handlerPlugin(pluginData));
                } else if (dataType == DataEventTypeEnum.DELETE) {
                    // 插件配置数据的删除
                    BaseDataCache.getInstance().removePluginData(pluginData);
                    Optional.ofNullable(handlerMap.get(pluginData.getName())).ifPresent(handler -> handler.removePlugin(pluginData));
                }
            } else if (data instanceof SelectorData) {
                SelectorData selectorData = (SelectorData) data;
                if (dataType == DataEventTypeEnum.UPDATE) {
                    // 选择器数据的更新
                    BaseDataCache.getInstance().cacheSelectData(selectorData);
                    Optional.ofNullable(handlerMap.get(selectorData.getPluginName())).ifPresent(handler -> handler.handlerSelector(selectorData));
                } else if (dataType == DataEventTypeEnum.DELETE) {
                    // 选择器数据的删除
                    BaseDataCache.getInstance().removeSelectData(selectorData);
                    Optional.ofNullable(handlerMap.get(selectorData.getPluginName())).ifPresent(handler -> handler.removeSelector(selectorData));
                }
            } else if (data instanceof RuleData) {
                RuleData ruleData = (RuleData) data;
                if (dataType == DataEventTypeEnum.UPDATE) {
                    // 规则数据的更新
                    BaseDataCache.getInstance().cacheRuleData(ruleData);
                    Optional.ofNullable(handlerMap.get(ruleData.getPluginName())).ifPresent(handler -> handler.handlerRule(ruleData));
                } else if (dataType == DataEventTypeEnum.DELETE) {
                    // 规则数据的删除
                    BaseDataCache.getInstance().removeRuleData(ruleData);
                    Optional.ofNullable(handlerMap.get(ruleData.getPluginName())).ifPresent(handler -> handler.removeRule(ruleData));
                }
            }
        });
    }
}
```

&ensp;&ensp;&ensp;&ensp;在上面的代码中，我们看它又对相关数据的更新做了一层封装，在其中我们找到一点命名不是太好的地方

&ensp;&ensp;&ensp;&ensp;下面我们来看下这些：onSelectorSubscribe 之类的函数在什么地方被调用了，通过Alt+F7发现（排查测试类和本身），有下面的三个模块进行引用：

- soul-sync-data-http : PluginDataRefresh
- soul-sync-data-nacos : NacosCacheHandler
- soul-sync-data-websocket : PluginDataHandler
- soul-sync-data-zookeeper : ZookeeperSyncDataService

&ensp;&ensp;&ensp;&ensp;结合前面的了解，可以看到是四大同步方式模块：HTTP、Nacos、websocket、zookeeper

&ensp;&ensp;&ensp;&ensp;大致浏览了一下，这思路同步模块的代码差异比较大，那就是四篇的分析文章了啊

## 总结
&ensp;&ensp;&ensp;&ensp;此篇文章大致梳理了数据同步比较基础的类：

- BaseDataCache : 配置缓存，提供更新、删除、查询
- CommonPluginDataSubscriber : 对BaseDataCache的封装，提供更新和删除
- 同步模块：调用CommonPluginDataSubscriber的接口进行数据同步
    - soul-sync-data-http : PluginDataRefresh
    - soul-sync-data-nacos : NacosCacheHandler
    - soul-sync-data-websocket : PluginDataHandler
    - soul-sync-data-zookeeper : ZookeeperSyncDataService

&ensp;&ensp;&ensp;&ensp;此篇文章先对数据同步有个大致的了解，下面的文章开始进行四大同步模块的分析