# Soul网关源码阅读（十三）Websocket同步数据-Bootstrap端
***
## 简介
&ensp;&ensp;&ensp;&ensp;此篇文章，我们来探索下Websocket数据同步具体细节

## 示例运行
&ensp;&ensp;&ensp;&ensp;启动数据库：

```shell script
docker run --name mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 -d mysql:latest
```

&ensp;&ensp;&ensp;&ensp;首先运行Soul-Admin

&ensp;&ensp;&ensp;&ensp;配置Soul-Bootstrap，配置websocket同步方式，运行Soul-Bootstrap，大致如下：

```xml
soul :
    file:
      enabled: true
    corss:
      enabled: true
    dubbo :
      parameter: multi
    # 把 websocket 数据同步打开
    sync:
        websocket :
             urls: ws://localhost:9095/websocket
```

&ensp;&ensp;&ensp;&ensp;运行Soul-Example-HTTP，注册一些数据用于Debug测试

## 源码Debug
&ensp;&ensp;&ensp;&ensp;在上篇的分析中，我们知道了数据更新的核心类及其核心代码如下：

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

&ensp;&ensp;&ensp;&ensp;我们先拉看一手插件配置数据，在上面的：onSubscribe 和 unSubscribe 打上断点，重启Soul-Bootstrap

```java
public class PluginDataHandler extends AbstractDataHandler<PluginData> {

    // 恢复，启动的时候执行
    @Override
    protected void doRefresh(final List<PluginData> dataList) {
        // 清空缓存数据
        pluginDataSubscriber.refreshPluginDataSelf(dataList);
        dataList.forEach(pluginDataSubscriber::onSubscribe);
    }

    // 数据更新（增改）
    @Override
    protected void doUpdate(final List<PluginData> dataList) {
        dataList.forEach(pluginDataSubscriber::onSubscribe);
    }

    // 数据删除
    @Override
    protected void doDelete(final List<PluginData> dataList) {
        dataList.forEach(pluginDataSubscriber::unSubscribe);
    }
}
```

&ensp;&ensp;&ensp;&ensp;这个类就是websocket同步模块中唯一调用CommonPluginDataSubscriber相关接口的，可以看到它就三个接口：恢复、更新、删除

&ensp;&ensp;&ensp;&ensp;datalist我们查看值，可以发现全是我们插件相关的数据，很重要，我们继续跟踪其来源

&ensp;&ensp;&ensp;&ensp;我们接着查看调用栈，来到下面的函数，看到它对应调用了前面函数的三个接口

```java
public abstract class AbstractDataHandler<T> implements DataHandler {

    // 对应上面的三个操作：恢复、更新、删除
    @Override
    public void handle(final String json, final String eventType) {
        List<T> dataList = convert(json);
        if (CollectionUtils.isNotEmpty(dataList)) {
            DataEventTypeEnum eventTypeEnum = DataEventTypeEnum.acquireByName(eventType);
            switch (eventTypeEnum) {
                case REFRESH:
                case MYSELF:
                    doRefresh(dataList);
                    break;
                case UPDATE:
                case CREATE:
                    doUpdate(dataList);
                    break;
                case DELETE:
                    doDelete(dataList);
                    break;
                default:
                    break;
            }
        }
    }
}
```

&ensp;&ensp;&ensp;&ensp;我们查看下DataEventTypeEnum:DELETE/CREATE/UPDATE/REFRESH/MYSELF,基本能对应上下面的SWITH的case

&ensp;&ensp;&ensp;&ensp;而且看到json转成的dataList，我们继续跟

```java
public class WebsocketDataHandler {

    public void executor(final ConfigGroupEnum type, final String json, final String eventType) {
        ENUM_MAP.get(type).handle(json, eventType);
    }
}
```

&ensp;&ensp;&ensp;&ensp;这个函数type，和evenTtype稍微注意下，看一看ConfigGroupEnum:APP_AUTH/PLUGIN/RULE/SELECTOR/META_DATA,找了一些前面我们忽略的同步数据，发现除了插件配置数据、选择器数据、规则数据，还有APP_AUTH数据和META_DATA（感觉和RPC相关），后面我们补上测一测

&ensp;&ensp;&ensp;&ensp;继续跟，来到属性的Websocket，和前面分析文章中的基本一致，都是通过起一个websocket客户端，接收到消息后触发调用

```java
@Slf4j
public final class SoulWebsocketClient extends WebSocketClient {
    
    @Override
    public void onMessage(final String result) {
        handleResult(result);
    }
    
    @SuppressWarnings("ALL")
    private void handleResult(final String result) {
        WebsocketData websocketData = GsonUtils.getInstance().fromJson(result, WebsocketData.class);
        ConfigGroupEnum groupEnum = ConfigGroupEnum.acquireByName(websocketData.getGroupType());
        String eventType = websocketData.getEventType();
        String json = GsonUtils.getInstance().toJson(websocketData.getData());
        websocketDataHandler.executor(groupEnum, json, eventType);
    }
}
```

&ensp;&ensp;&ensp;&ensp;基本上一个初始化的流程是更新完毕了

&ensp;&ensp;&ensp;&ensp;后面我们测试下修改，在后面管理界面，把rate_limiter状态给关了，修改后立马进入了debug，收到的数据如下：

```json
{"groupType":"PLUGIN","eventType":"UPDATE","data":[{"id":"4","name":"rate_limiter","config":"{\"mode\":\"standalone\",\"master\":\"mymaster\",\"url\":\"127.0.0.1:6379\"}","role":1,"enabled":false}]}
```

&ensp;&ensp;&ensp;&ensp;可以看到比较关键的数据：groupType、eventType、data等等

&ensp;&ensp;&ensp;&ensp;后面进行了删除的测试，流程基本一致，类型变了而已

&ensp;&ensp;&ensp;&ensp;后面进行其他类型的测试：PLUGIN/RULE/SELECTOR/META_DATA,APP_AUTH不太去确定，就没有测

&ensp;&ensp;&ensp;&ensp;总体而言还是比较简单清晰，那就直接总结一波

## 总结
&ensp;&ensp;&ensp;&ensp;总体来说Websocket同步还是比较简单的,可能让人疑惑的是Websocket的使用吧。由于在工作中还是比较常使用websocket，所以理解读数据那块还是比较轻松的，如果大家有疑惑的话，可以搜索Spring Websocket的使用教程，自己动手玩一玩，估计就可以了

&ensp;&ensp;&ensp;&ensp;总结今天的Websocket同步如下图：

![](./picture/websocket-bootstrap.png)

&ensp;&ensp;&ensp;&ensp;如上图所示，数据更新的流程如下：

- 1.SoulWebsocketClient ：从Soul-Admin接收数据，进行更新，里面有数据类型、操作类型还有具体信息
- 2.WebsocketDataHandler ：根据这ConfigGroupEnum（数据类型），调用相应的subscribe
- 3.subscribe ：具体subscribe调用相应的类型的DataCache进行数据更新和删除

