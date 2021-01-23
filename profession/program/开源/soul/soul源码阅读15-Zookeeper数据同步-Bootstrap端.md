# Soul网关源码阅读（十五）Zookeeper数据同步-Bootstrap端
***
## 简介
&ensp;&ensp;&ensp;&ensp;本篇文章对Soul网关Bootstrap的Zookeeper数据同步进行探索

## 示例启动
### 环境配置
&ensp;&ensp;&ensp;&ensp;使用docker启动mysql和zk

```shell script
docker run -dit --name zk -p 2181:2181 zookeepe
docker run --name mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 -d mysql:latest
```

### Soul-Admin配置
&ensp;&ensp;&ensp;&ensp;启动时候，我们先把Soul-admin和Soul-Bootstrap的Websocket同步给关了，开启HTTP同步，配置大致如下：

&ensp;&ensp;&ensp;&ensp;Soul-Admin的HTTP同步方式配置，暂时把其他同步方式给注释关闭

```xml
  sync:
      zookeeper:
          url: localhost:2181
          sessionTimeout: 5000
          connectionTimeout: 2000
```

### Soul-Bootstrap配置
&ensp;&ensp;&ensp;&ensp;Soul-Bootstrap的Http同步方式配置，暂时把其他同步方式给注释关闭

```xml
soul :
    file:
      enabled: true
    corss:
      enabled: true
    dubbo :
      parameter: multi
    sync:
        zookeeper:
             url: localhost:2181
             sessionTimeout: 5000
             connectionTimeout: 2000
```

### 启动失败
&ensp;&ensp;&ensp;&ensp;启动Soul-admin、Soul-Bootstrap、Soul-Example-HTTP

&ensp;&ensp;&ensp;&ensp;http://localhost:9195/dubbo/findById?id=1 ,访问链接失败

&ensp;&ensp;&ensp;&ensp;进行debug的时候发现没有数据，只用变更的时候会发数据过来，重启了试试不行

&ensp;&ensp;&ensp;&ensp;感觉可能是bug，但是首先排除下自己的代码版本问题和自己的使用问题，问了问群里的老哥，发现是自己的使用问题

### 启动修复
&ensp;&ensp;&ensp;&ensp;解决的办法是拉一次自己的代码，保证是最新的，然后删除zookeeper里面的soul数据，重启Admin和Bootstrap，重新访问，成功

&ensp;&ensp;&ensp;&ensp;下面是zk图形化客户端，用起来比价方便，win也可以使用（win下第一次启动可能显示不全，关闭再启动即可），下面是搜索到的安装教程和下载链接：

- [zookeeper图形化客户端工具ZooInspector的使用](https://blog.csdn.net/kxj19980524/article/details/86840558)
- https://issues.apache.org/jira/secure/attachment/12436620/ZooInspector.zip

## 源码查看
&ensp;&ensp;&ensp;&ensp;我们直接定位到[Soul网关源码阅读（十二）数据同步初探](https://juejin.cn/post/6920596173925384206)中的zk同步的入口类： ZookeeperSyncDataService

&ensp;&ensp;&ensp;&ensp;稍微用过zk的，就可以看到熟悉的watch，监听数据的变化，从下面的代码大致可以看出监听我前面分析的五种数据：插件数据、选择器数据、规则数据、AppAuth数据、Metadata数据

```java
public class ZookeeperSyncDataService implements SyncDataService, AutoCloseable {

    public ZookeeperSyncDataService(final ZkClient zkClient, final PluginDataSubscriber pluginDataSubscriber,
                                    final List<MetaDataSubscriber> metaDataSubscribers, final List<AuthDataSubscriber> authDataSubscribers) {
        this.zkClient = zkClient;
        this.pluginDataSubscriber = pluginDataSubscriber;
        this.metaDataSubscribers = metaDataSubscribers;
        this.authDataSubscribers = authDataSubscribers;
        watcherData();
        watchAppAuth();
        watchMetaData();
    }

    private void watcherAll(final String pluginName) {
        watcherPlugin(pluginName);
        watcherSelector(pluginName);
        watcherRule(pluginName);
    }
}
```

&ensp;&ensp;&ensp;&ensp;我们在构造函数上打上断点，看看调用栈，发现是和HTTP同步差不多的，如下代码所示，可以看到使用Spring方式启动

```java
public class ZookeeperSyncDataConfiguration {
   
    @Bean
    public SyncDataService syncDataService(final ObjectProvider<ZkClient> zkClient, final ObjectProvider<PluginDataSubscriber> pluginSubscriber,
                                           final ObjectProvider<List<MetaDataSubscriber>> metaSubscribers, final ObjectProvider<List<AuthDataSubscriber>> authSubscribers) {
        log.info("you use zookeeper sync soul data.......");
        return new ZookeeperSyncDataService(zkClient.getIfAvailable(), pluginSubscriber.getIfAvailable(),
                metaSubscribers.getIfAvailable(Collections::emptyList), authSubscribers.getIfAvailable(Collections::emptyList));
    }

    @Bean
    public ZkClient zkClient(final ZookeeperConfig zookeeperConfig) {
        return new ZkClient(zookeeperConfig.getUrl(), zookeeperConfig.getSessionTimeout(), zookeeperConfig.getConnectionTimeout());
    }

}
```

&ensp;&ensp;&ensp;&ensp;简单的调试和看了看，发现zk的同步方式和websocket一样还是比较简单的，结构也是非常的清晰，这里就不做过多的介绍了，就说明下其核心逻辑，代码如下：

&ensp;&ensp;&ensp;&ensp;构造函数中启动多所有数据的监听，并进行五种树节点初始化

&ensp;&ensp;&ensp;&ensp;需要注意理解的是对列表数据的监听（监听增删之类），监听列表中单个数据的变化，代码中的监听基本都是这种套路，注意理解即可

```java
public class ZookeeperSyncDataService implements SyncDataService, AutoCloseable {

    public ZookeeperSyncDataService(final ZkClient zkClient, final PluginDataSubscriber pluginDataSubscriber,
                                    final List<MetaDataSubscriber> metaDataSubscribers, final List<AuthDataSubscriber> authDataSubscribers) {
        this.zkClient = zkClient;
        this.pluginDataSubscriber = pluginDataSubscriber;
        this.metaDataSubscribers = metaDataSubscribers;
        this.authDataSubscribers = authDataSubscribers;
        // 下面三个函数起到了初始化和启动监听的作用
        watcherData();
        watchAppAuth();
        watchMetaData();
    }

    private void watcherData() {
        final String pluginParent = ZkPathConstants.PLUGIN_PARENT;
        // 获取插件列表，启动监听（在其中同时进行插件数据、选择器、规则的初始化）
        List<String> pluginZKs = zkClientGetChildren(pluginParent);
        for (String pluginName : pluginZKs) {
            watcherAll(pluginName);
        }
        
        // 启动监听，数据更新（新增和修改），对更新的插件进行监听
        zkClient.subscribeChildChanges(pluginParent, (parentPath, currentChildren) -> {
            if (CollectionUtils.isNotEmpty(currentChildren)) {
                for (String pluginName : currentChildren) {
                    watcherAll(pluginName);
                }
            }
        });
    }

    private void watcherAll(final String pluginName) {
        watcherPlugin(pluginName);
        watcherSelector(pluginName);
        watcherRule(pluginName);
    }

    private void watcherPlugin(final String pluginName) {
        String pluginPath = ZkPathConstants.buildPluginPath(pluginName);
        if (!zkClient.exists(pluginPath)) {
            zkClient.createPersistent(pluginPath, true);
        }
        // 初始化插件数据
        cachePluginData(zkClient.readData(pluginPath));
        // 监听插件数据
        subscribePluginDataChanges(pluginPath, pluginName);
    }

    // 选择器结构大致是：插件 --> 选择器列表 --> 单个选择器
    // 所有最终需要对单个选择器进行初始化和监听
    // 并对上层的选择器列表进行监听，列表也会更新
    private void watcherSelector(final String pluginName) {
        String selectorParentPath = ZkPathConstants.buildSelectorParentPath(pluginName);
        List<String> childrenList = zkClientGetChildren(selectorParentPath);
        if (CollectionUtils.isNotEmpty(childrenList)) {
            childrenList.forEach(children -> {
                String realPath = buildRealPath(selectorParentPath, children);
                cacheSelectorData(zkClient.readData(realPath));
                subscribeSelectorDataChanges(realPath);
            });
        }
        subscribeChildChanges(ConfigGroupEnum.SELECTOR, selectorParentPath, childrenList);
    }

    // 原理同选择器
    private void watcherRule(final String pluginName) {
        String ruleParent = ZkPathConstants.buildRuleParentPath(pluginName);
        List<String> childrenList = zkClientGetChildren(ruleParent);
        if (CollectionUtils.isNotEmpty(childrenList)) {
            childrenList.forEach(children -> {
                String realPath = buildRealPath(ruleParent, children);
                cacheRuleData(zkClient.readData(realPath));
                subscribeRuleDataChanges(realPath);
            });
        }
        subscribeChildChanges(ConfigGroupEnum.RULE, ruleParent, childrenList);
    }

    // 原理类似选择器
    private void watchAppAuth() {
        final String appAuthParent = ZkPathConstants.APP_AUTH_PARENT;
        List<String> childrenList = zkClientGetChildren(appAuthParent);
        if (CollectionUtils.isNotEmpty(childrenList)) {
            childrenList.forEach(children -> {
                String realPath = buildRealPath(appAuthParent, children);
                cacheAuthData(zkClient.readData(realPath));
                subscribeAppAuthDataChanges(realPath);
            });
        }
        subscribeChildChanges(ConfigGroupEnum.APP_AUTH, appAuthParent, childrenList);
    }

    // 原理类似选择器
    private void watchMetaData() {
        final String metaDataPath = ZkPathConstants.META_DATA;
        List<String> childrenList = zkClientGetChildren(metaDataPath);
        if (CollectionUtils.isNotEmpty(childrenList)) {
            childrenList.forEach(children -> {
                String realPath = buildRealPath(metaDataPath, children);
                cacheMetaData(zkClient.readData(realPath));
                subscribeMetaDataChanges(realPath);
            });
        }
        subscribeChildChanges(ConfigGroupEnum.APP_AUTH, metaDataPath, childrenList);
    }
}
```

## 总结
&ensp;&ensp;&ensp;&ensp;总体来说，zookeeper同步方式还是比较容易理解的，代码结构也清晰

&ensp;&ensp;&ensp;&ensp;可能没有用过zookeeper的会有点疑惑，这里就简单介绍下自己的理解：

&ensp;&ensp;&ensp;&ensp;上面分析中提到的监听其核心是：发布-订阅

&ensp;&ensp;&ensp;&ensp;zk客户端连接上zk服务端，当服务端的数据发送变化的时候，服务端会发送变化的数据给客户端，客户端收到变化的数据后进行操作

&ensp;&ensp;&ensp;&ensp;这些变化的数据可以是目录（上面提到的列表数据），也可以是具体单个文件（列表中具体数据）

&ensp;&ensp;&ensp;&ensp;理解了这个，大致就能理解zk同步的核心了

&ensp;&ensp;&ensp;&ensp;总结下zookeeper的同步流程大致如下：

- 1.读取初始数据进行初始化
- 2.对读取的初始化数据进行监听
- 3.读取变化的数据进行本地缓存更新，并对其进行监听
