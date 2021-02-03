# Soul网关源码解析（二十七）服务注册与下线
***

http://localhost:9195/dubbo/findById?id=1

```json
{
    "code": -107,
    "message": "Can not find selector, please check your configuration!",
    "data": null
}
```

```json
{
    "code": 500,
    "message": "Internal Server Error",
    "data": "No provider available from registry localhost:2181 for service org.dromara.soul.examples.dubbo.api.service.DubboTestService on consumer 172.21.160.1 use dubbo version 2.6.5, please check status of providers(disabled, not registered or in blacklist)."
}
```

```json
{
    "code": -107,
    "message": "Can not find selector, please check your configuration!",
    "data": null
}
```

```java
public class AlibabaDubboMetaDataSubscriber implements MetaDataSubscriber {

    private static final ConcurrentMap<String, MetaData> META_DATA = Maps.newConcurrentMap();

    @Override
    public void onSubscribe(final MetaData metaData) {
        if (RpcTypeEnum.DUBBO.getName().equals(metaData.getRpcType())) {
            MetaData exist = META_DATA.get(metaData.getPath());
            if (Objects.isNull(META_DATA.get(metaData.getPath())) || Objects.isNull(ApplicationConfigCache.getInstance().get(metaData.getPath()))) {
                // The first initialization
                ApplicationConfigCache.getInstance().initRef(metaData);
            } else {
                // There are updates, which only support the update of four properties of serviceName rpcExt parameterTypes methodName,
                // because these four properties will affect the call of Dubbo;
                if (!metaData.getServiceName().equals(exist.getServiceName())
                        || !metaData.getRpcExt().equals(exist.getRpcExt())
                        || !metaData.getParameterTypes().equals(exist.getParameterTypes())
                        || !metaData.getMethodName().equals(exist.getMethodName())) {
                    ApplicationConfigCache.getInstance().build(metaData);
                }
            }
            META_DATA.put(metaData.getPath(), metaData);
        }
    }

    @Override
    public void unSubscribe(final MetaData metaData) {
        if (RpcTypeEnum.DUBBO.getName().equals(metaData.getRpcType())) {
            ApplicationConfigCache.getInstance().invalidate(metaData.getPath());
            META_DATA.remove(metaData.getPath());
        }
    }
}
```

http://127.0.0.1:9195/http/order/findById?id=1111

```json
{
    "code": -107,
    "message": "Can not find selector, please check your configuration!",
    "data": null
}
```

```json
{
    "code": 500,
    "message": "Internal Server Error",
    "data": "Connection refused: no further information: /172.21.160.1:8188"
}
```

下线的情况是否可以分为三种：

- 整个应用服务下线：selector 和 rule 都需要删除
- selector下线（一台服务器其两个web服务，用了同一个context）：删除selector和rule（其实情况和第一个是一样的，因为一个应用只能用一个context
  - 经过测试，一台服务器中的两个不同的应用可以用同一个context，且能注入到同一个selector中，视为负载均衡的服务器，但两个服务器其实有自己独特的路径，会导致访问异常
  - 也就是说，上面这种情况的配置是不予许存在的，Context（selector）应该具有唯一性，要么是单台，要么是集群的负载均衡
- rule下线，情况同selector，这种是目前是不能存在的

综上所述：服务下线应该是整个应用的下线，单个的服务的下线。所以下线的时候需要知道selector和相应的ip和port（因为集群的存在）

HTTP的下线应该是上面那样的

RPC的下线，比如Dubbo，第三方的做了相应的工作，可能应用已经下线，但还是要匹配成功，进入调用后才能报Provider Error之类的
查看其Dubbo的Selector的配置，没有进一步的标识区分，也就是，如果是Dubbo集群，有一个Selector(Context)下线了，还不能进行删除（两个ZK，创建了同一个临时节点，其中一个挂了，节点会删除吗？）

zk建立节点，如果同一个服务器，那路径会是一样的，路径需要再进行区分（需要再加一层IP+端口进行标识）

selector：有集群概念
    - 增 ： selector（context）+ ip-port 作为唯一标识
    - 删除 ：
      - ip-port : 集群中的某一台挂了，删除负载均衡中的某一个（根据ip+port）；
        - 其中的子节点数据发生变化，那就是rule进行了更新
          - 如果子节点为0，则说明改服务器已经挂了，需要移除divide中的对应的ip+port的集群节点
          - 增加或者删除的话（两个以上才会出现），那只能进行更改rule（那集群提供的服务在负载均衡时有可能打错位置，rule并不是所有的机器都一样）
      - selector ： 如果全部的都挂了，需要删除对一个的selector（如果selector是临时节点，是第一台启动时建立的，随后启动第二台，第一台挂了以后，是否会删除selector？由于临时节点不能接子节点，并且需要从节点中读取DTO数据，所以只能数据节点是临时子节点，其他都为持久性的节点）


ZK临时节点和持久节点的区别是?
语法不同 create key value 是持久节点，零时节点需要加 -e 的参数

临时节点和客户端断开连接自动删除，持久节点不会

临时节点下面不能接子节点，持久节点可以