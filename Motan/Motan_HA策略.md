## Motan系列文章

* [Motan如何完成与Spring的集成](https://github.com/Lord-X/awesome-it-blog/blob/master/Motan/Motan_%E5%A6%82%E4%BD%95%E5%AE%8C%E6%88%90%E4%B8%8ESpring%E7%9A%84%E9%9B%86%E6%88%90.md)
* [Motan的SPI插件扩展机制](https://github.com/Lord-X/awesome-it-blog/blob/master/Motan/Motan_SPI%E6%8F%92%E4%BB%B6%E6%89%A9%E5%B1%95%E6%9C%BA%E5%88%B6.md)
* [Motan服务注册](https://github.com/Lord-X/awesome-it-blog/blob/master/Motan/Motan_%E6%9C%8D%E5%8A%A1%E6%B3%A8%E5%86%8C.md)
* [Motan服务调用](https://github.com/Lord-X/awesome-it-blog/blob/master/Motan/Motan_%E6%9C%8D%E5%8A%A1%E8%B0%83%E7%94%A8.md)
* [Motan心跳机制](https://github.com/Lord-X/awesome-it-blog/blob/master/Motan/Motan_%E5%BF%83%E8%B7%B3%E6%9C%BA%E5%88%B6.md)
* [Motan负载均衡策略](https://github.com/Lord-X/awesome-it-blog/blob/master/Motan/Motan_LoadBalance.md)
* [Motan高可用策略](https://github.com/Lord-X/awesome-it-blog/blob/master/Motan/Motan_HA%E7%AD%96%E7%95%A5.md)

---

HA即高可用，也可以理解为容错策略。在Motan中，HA作用于Client端，其含义是当RPC调用失败时，采取的策略。目前有以下两种策略：

* Failover：失效转移（默认）

配置方式如下：

```text
<motan:protocol ... haStrategy="failover"/>
```

其含义是，当RPC调用失败时，自动重试其他服务器。

* Failfast：快速失败

配置方式如下：

```text
<motan:protocol ... haStrategy="failfast"/>
```

其含义是，只发起一次调用，失败立即报错。

### 0 工程结构及继承关系

下图展示了HA源码在工程中的位置：

![](http://image.feathers.top/image/Motan_HA1.png)

下面来看一下HA的体系结构：

![](http://image.feathers.top/image/Motan_HA2.png)

这里最核心的是 `call()` 方法。这个方法在 `HaStrategy` 接口定义，由具体实现类实现，用于完成HA策略逻辑。

### 1 HA策略的选取及设置

Motan会在Cluster初始化阶段设置HA策略。HA策略可通过上述配置来定义，如果不配置，默认使用 `Failover`，即失效转移策略。

在实现上，HA采用插件化开发（即SPI），每个HA的具体实现都是一个SPI，其实现类都标注了 `@SpiMeta` 注解，在setHaStrategy时，根据具体的HA名称即可找到具体的实现。以 `Failover` 为例：

```java
@SpiMeta(name = "failover")
public class FailoverHaStrategy<T> extends AbstractHaStrategy<T> {
    
}
```

然后在 `ClusterSupport` 中通过SPI获取HA的具体实现，并setHaStrategy。

```java
private void prepareCluster() {
    String clusterName = url.getParameter(URLParamType.cluster.getName(), URLParamType.cluster.getValue());
    String loadbalanceName = url.getParameter(URLParamType.loadbalance.getName(), URLParamType.loadbalance.getValue());
    // 获取HA策略，优选获取用户配置的，如果没有配置，取URLParamType配置的默认值，即 failover
    String haStrategyName = url.getParameter(URLParamType.haStrategy.getName(), URLParamType.haStrategy.getValue());

    cluster = ExtensionLoader.getExtensionLoader(Cluster.class).getExtension(clusterName);
    LoadBalance<T> loadBalance = ExtensionLoader.getExtensionLoader(LoadBalance.class).getExtension(loadbalanceName);
    // 通过SPI的方式获取具体HA实现
    HaStrategy<T> ha = ExtensionLoader.getExtensionLoader(HaStrategy.class).getExtension(haStrategyName);
    ha.setUrl(url);
    cluster.setLoadBalance(loadBalance); // setLoadBalance
    cluster.setHaStrategy(ha);
    cluster.setUrl(url);
}
```

### 2 调用时序

配置好HA以后，再来看一下调用时序。以下时序图说明了一个RPC请求的调用时序，主要关注HaStrategy的调用时机。

![Motan_Referer调用时序图](http://image.feathers.top/image/Motan_Referer调用时序图.jpg)

至此，本文已说明HA的设置以及HA的调用时机，下面来介绍一下两种HA的具体实现。

### 3 HA的实现

下面介绍一下两种HA的具体源码实现：

* Failover：失效转移
* Failfast：快速失败

#### 3.1 Failover

Failover是默认的HA策略。

```java
@SpiMeta(name = "failover")
public class FailoverHaStrategy<T> extends AbstractHaStrategy<T> {
    
    // 用ThreadLocal存储每次调用中 可用的集群备选节点
    protected ThreadLocal<List<Referer<T>>> referersHolder = new ThreadLocal<List<Referer<T>>>() {
        @Override
        protected java.util.List<com.weibo.api.motan.rpc.Referer<T>> initialValue() {
            return new ArrayList<Referer<T>>();
        }
    };

    @Override
    public Response call(Request request, LoadBalance<T> loadBalance) {

        List<Referer<T>> referers = selectReferers(request, loadBalance);
        if (referers.isEmpty()) {
            throw new MotanServiceException(String.format("FailoverHaStrategy No referers for request:%s, loadbalance:%s", request,
                    loadBalance));
        }
        URL refUrl = referers.get(0).getUrl();
        // 获取重试次数配置，默认等于0，即不重试
        int tryCount =
                refUrl.getMethodParameter(request.getMethodName(), request.getParamtersDesc(), URLParamType.retries.getName(),
                        URLParamType.retries.getIntValue());
        // 如果有问题，则设置为不重试
        if (tryCount < 0) {
            tryCount = 0;
        }

        for (int i = 0; i <= tryCount; i++) {
            // 失效转移，当tryCount大于0时，会挨个尝试referers中的节点。
            Referer<T> refer = referers.get(i % referers.size());
            try {
                // 设置重试次数
                request.setRetries(i);
                return refer.call(request);
            } catch (RuntimeException e) {
                // 对于业务异常，直接抛出
                if (ExceptionUtil.isBizException(e)) {
                    throw e;
                // 当重试次数未达到tryCount时，继续下一次循环，否则抛出异常
                } else if (i >= tryCount) {
                    throw e;
                }
                LoggerUtil.warn(String.format("FailoverHaStrategy Call false for request:%s error=%s", request, e.getMessage()));
            }
        }

        throw new MotanFrameworkException("FailoverHaStrategy.call should not come here!");
    }
    
    // 根据LoadBalance策略选取备选节点，并存在ThreadLocal中
    protected List<Referer<T>> selectReferers(Request request, LoadBalance<T> loadBalance) {
        List<Referer<T>> referers = referersHolder.get();
        referers.clear();
        loadBalance.selectToHolder(request, referers);
        return referers;
    }

}
```

由上述代码可知，失效转移机制和 `retries` 这个参数有关，当这个参数为0时，只会调用一次，如果失败就直接抛出异常了，当这个参数大于0时，会挨个尝试LoadBalance策略返回的可用集群节点。

#### 3.2 Failfast

Failfast会在RPC调用失败时直接抛出异常，其实现很简单：

```java
@SpiMeta(name = "failfast")
public class FailfastHaStrategy<T> extends AbstractHaStrategy<T> {

    @Override
    public Response call(Request request, LoadBalance<T> loadBalance) {
        Referer<T> refer = loadBalance.select(request);
        return refer.call(request);
    }
}
```

由于快速失败策略不用重试，所以调用LoadBalance的 `select` 方法选取一个节点即可（Failover因为需要满足失效转移，所以需要多个备选节点，`referersHolder` 就是获取多个节点的方法），如果节点调用失败，则抛出 RuntimeException。

## 参考

* [Motan_Github](https://github.com/weibocom/motan/wiki/zh_userguide#%E5%9F%BA%E6%9C%AC%E4%BB%8B%E7%BB%8D)