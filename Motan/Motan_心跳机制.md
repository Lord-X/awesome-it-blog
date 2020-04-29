## Motan系列文章

* [Motan如何完成与Spring的集成](https://github.com/Lord-X/awesome-it-blog/blob/master/Motan/Motan_%E5%A6%82%E4%BD%95%E5%AE%8C%E6%88%90%E4%B8%8ESpring%E7%9A%84%E9%9B%86%E6%88%90.md)
* [Motan的SPI插件扩展机制](https://github.com/Lord-X/awesome-it-blog/blob/master/Motan/Motan_SPI%E6%8F%92%E4%BB%B6%E6%89%A9%E5%B1%95%E6%9C%BA%E5%88%B6.md)
* [Motan服务注册](https://github.com/Lord-X/awesome-it-blog/blob/master/Motan/Motan_%E6%9C%8D%E5%8A%A1%E6%B3%A8%E5%86%8C.md)
* [Motan服务调用](https://github.com/Lord-X/awesome-it-blog/blob/master/Motan/Motan_%E6%9C%8D%E5%8A%A1%E8%B0%83%E7%94%A8.md)
* [Motan心跳机制](https://github.com/Lord-X/awesome-it-blog/blob/master/Motan/Motan_%E5%BF%83%E8%B7%B3%E6%9C%BA%E5%88%B6.md)
* [Motan负载均衡策略](https://github.com/Lord-X/awesome-it-blog/blob/master/Motan/Motan_LoadBalance.md)
* [Motan高可用策略](https://github.com/Lord-X/awesome-it-blog/blob/master/Motan/Motan_HA%E7%AD%96%E7%95%A5.md)

---

Motan的心跳机制分为两种：
* Provider到Registry的心跳监测
* Consumer到Provider的心跳监测

下面分别来介绍这两种监测的实现。

### 0 Provider到Registry的心跳监测

这里将以Zookeeper为注册中心为例，说明其心跳监测的实现。

Provider到Registry的心跳监测是在 `服务注册` 这个过程中实现的，服务注册的过程在 [Motan服务注册](https://github.com/Lord-X/awesome-it-blog/blob/master/Motan/Motan_%E6%9C%8D%E5%8A%A1%E6%B3%A8%E5%86%8C.md) 一文中已详细描述，这里不再赘述。

具体来说，是在Provider启动完成后，向Registry注册时实现的。这个过程首先会根据Registry的URL拿到对应的Registry对象，具体代码在 `AbstractRegistryFactory` 类的 `getRegistry(URL url)` 方法中实现。

```java
@Override
public Registry getRegistry(URL url) {
    String registryUri = getRegistryUri(url);
    try {
        lock.lock();
        // 如果已经创建过，直接拿已经创建好的实例，否则调用下面的 createRegistry 方法创建新的Registry实例
        Registry registry = registries.get(registryUri);
        if (registry != null) {
            return registry;
        }
        // 心跳监测机制在这里实现
        registry = createRegistry(url);
        if (registry == null) {
            throw new MotanFrameworkException("Create registry false for url:" + url, MotanErrorMsgConstant.FRAMEWORK_INIT_ERROR);
        }
        registries.put(registryUri, registry);
        return registry;
    } catch (Exception e) {
        throw new MotanFrameworkException("Create registry false for url:" + url, e, MotanErrorMsgConstant.FRAMEWORK_INIT_ERROR);
    } finally {
        lock.unlock();
    }
}
```

现在的问题就是看一下他具体是怎么创建Registry的了。

`createRegistry` 是个模板方法，具体实现在其子类中，以ZK来说，其具体实现在 `ZookeeperRegistryFactory` 类中。具体的 `createRegistry` 实现如下：

```java
public class ZookeeperRegistryFactory extends AbstractRegistryFactory {
    @Override
    protected Registry createRegistry(URL registryUrl) {
        try {
            // 通过ZKClient链接ZK
            int timeout = registryUrl.getIntParameter(URLParamType.connectTimeout.getName(), URLParamType.connectTimeout.getIntValue());
            int sessionTimeout = registryUrl.getIntParameter(URLParamType.registrySessionTimeout.getName(), URLParamType.registrySessionTimeout.getIntValue());
            ZkClient zkClient = createInnerZkClient(registryUrl.getParameter("address"), sessionTimeout, timeout);
            // 创建 Registry 实例
            return new ZookeeperRegistry(registryUrl, zkClient);
        } catch (ZkException e) {
            LoggerUtil.error("[ZookeeperRegistry] fail to connect zookeeper, cause: " + e.getMessage());
            throw e;
        }
    }

    protected ZkClient createInnerZkClient(String zkServers, int sessionTimeout, int connectionTimeout) {
        return new ZkClient(zkServers, sessionTimeout, connectionTimeout);
    }
}
```

最终通过new一个 `ZookeeperRegistry` 类的实例创建Registry，进到这个构造方法后，会通过super一层一层往父类中调。其他的过程暂且不谈，最终会调用到 `AbstractRegistry` 类的构造方法。具体的心跳监测就是在这里实现了。

```java
public AbstractRegistry(URL url) {
    this.registryUrl = url.createCopy();
    // register a heartbeat switcher to perceive service state change and change available state
    MotanSwitcherUtil.initSwitcher(MotanConstants.REGISTRY_HEARTBEAT_SWITCHER, false);
    MotanSwitcherUtil.registerSwitcherListener(MotanConstants.REGISTRY_HEARTBEAT_SWITCHER, new SwitcherListener() {

        @Override
        public void onValueChanged(String key, Boolean value) {
            if (key != null && value != null) {
                if (value) {
                    available(null);
                } else {
                    unavailable(null);
                }
            }
        }
    });
}
```

`MotanSwitcherUtil` 通过 `LocalSwitcherService` 类维护所有的Switcher状态，`MotanSwitcherUtil.initSwitcher` 的实现如下：

```java
public class MotanSwitcherUtil {
    private static SwitcherService switcherService = new LocalSwitcherService();

    public static void initSwitcher(String switcherName, boolean initialValue) {
        switcherService.initSwitcher(switcherName, initialValue);
    }
}
```

调用了 `LocalSwitcherService` 的 `initSwitcher` 方法，在 `LocalSwitcherService` 类中，维护了下面两个东西：

```java
// Switcher名字 -> Switcher
private static ConcurrentMap<String, Switcher> switchers = new ConcurrentHashMap<String, Switcher>();
// Switcher名字 -> 监听器列表
private ConcurrentHashMap<String, List<SwitcherListener>> listenerMap = new ConcurrentHashMap<>();
```

`switcherService.initSwitcher` 会调用serValue方法，实现如下：

```java
public void setValue(String switcherName, boolean value) {
    putSwitcher(new Switcher(switcherName, value));

    List<SwitcherListener> listeners = listenerMap.get(switcherName);
    if(listeners != null) {
        for (SwitcherListener listener : listeners) {
            listener.onValueChanged(switcherName, value);
        }
    }
}
```

`putSwitcher` 方法会将新创建的Switcher存到 `switchers` 中，对应的key为上面传进来的 `MotanConstants.REGISTRY_HEARTBEAT_SWITCHER` 常量，具体的值是:

```java
public static final String REGISTRY_HEARTBEAT_SWITCHER = "feature.configserver.heartbeat";
```

然后遍历监听器列表，触发 `onValueChanged` 方法。进行到现在，还没有向listenerMap注册监听器，所以这里没有监听器的调用。

此时我们有了一个Switcher了，是长这个样子的：

```java
public class Switcher {
    private boolean on = true;
    private String name; // 开关名
}
```

其中 name 就是 `feature.configserver.heartbeat` ，on 为 false。

然后回到最上面，开始注册监听器：`MotanSwitcherUtil.registerSwitcherListener`

里面最终会调用 `registerListener` 方法

```java
public void registerListener(String switcherName, SwitcherListener listener) {
    List listeners = Collections.synchronizedList(new ArrayList());
    List preListeners= listenerMap.putIfAbsent(switcherName, listeners);
    // 如果listenerMap已经存在switcherName这个key，直接将listener追加到list，否则创建新的list
    if (preListeners == null) {
        listeners.add(listener);
    } else {
        preListeners.add(listener);
    }
}
```

至此，Switcher和监听器的初始化工作就完成了，最后打开开关，调用 `available` 方法将server变为可用状态。`available` 会调用 `doAvailable` 方法将当前节点添加到ZK中的 `server` path下，并移除 `unavailableServer` path下的节点（如果存在）。

```java
protected void doAvailable(URL url) {
    try {
        serverLock.lock();
        if (url == null) {
            availableServices.addAll(getRegisteredServiceUrls());
            for (URL u : getRegisteredServiceUrls()) {
                removeNode(u, ZkNodeType.AVAILABLE_SERVER);
                removeNode(u, ZkNodeType.UNAVAILABLE_SERVER);
                createNode(u, ZkNodeType.AVAILABLE_SERVER);
            }
        } else {
            availableServices.add(url);
            removeNode(url, ZkNodeType.AVAILABLE_SERVER);
            removeNode(url, ZkNodeType.UNAVAILABLE_SERVER);
            createNode(url, ZkNodeType.AVAILABLE_SERVER);
        }
    } finally {
        serverLock.unlock();
    }
}
```