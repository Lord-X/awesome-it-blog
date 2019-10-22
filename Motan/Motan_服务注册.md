## Motan系列文章

* [Motan如何完成与Spring的集成](https://github.com/Lord-X/awesome-it-blog/blob/master/Motan/Motan_%E5%A6%82%E4%BD%95%E5%AE%8C%E6%88%90%E4%B8%8ESpring%E7%9A%84%E9%9B%86%E6%88%90.md)
* [Motan的SPI插件扩展机制](https://github.com/Lord-X/awesome-it-blog/blob/master/Motan/Motan_SPI%E6%8F%92%E4%BB%B6%E6%89%A9%E5%B1%95%E6%9C%BA%E5%88%B6.md)
* [Motan服务注册](https://github.com/Lord-X/awesome-it-blog/blob/master/Motan/Motan_%E6%9C%8D%E5%8A%A1%E6%B3%A8%E5%86%8C.md)
* [Motan服务调用](https://github.com/Lord-X/awesome-it-blog/blob/master/Motan/Motan_%E6%9C%8D%E5%8A%A1%E8%B0%83%E7%94%A8.md)

---

本文将以 `注解暴露服务` 的方式探究Motan服务的注册过程。

### 0 @MotanService注解是个啥

以 `@MotanService` 注解标记的类，在应用启动时，会被Motan扫描，并作为服务的具体实现注册到注册中心中。

就像下面这样：

```java
@MotanService(export = "demoMotan:8002")
public class MotanDemoServiceImpl implements MotanDemoService {

    @Override
    public String hello(String name) {
        System.out.println(name);
        return "Hello " + name + "!";
    }

    @Override
    public User rename(User user, String name) throws Exception {
        Objects.requireNonNull(user);
        System.out.println(user.getId() + " rename " + user.getName() + " to " + name);
        user.setName(name);
        return user;
    }
}
```

在 [Motan如何完成与Spring的集成](https://github.com/Lord-X/awesome-it-blog/blob/master/Motan/Motan_%E5%A6%82%E4%BD%95%E5%AE%8C%E6%88%90%E4%B8%8ESpring%E7%9A%84%E9%9B%86%E6%88%90.md) 一文中已经说过应用启动时，是如何扫描到 `@MotanService` 注解标记的类的，这里不再赘述。

### 1 @MotanService的解析

`@MotanService` 的解析过程在 `com.weibo.api.motan.config.springsupport.AnnotationBean` 类中的 `postProcessAfterInitialization(Object bean, String beanName)` 方法实现。

首先会解析配置信息，例如这个服务实现的接口是谁、以及`application`、`moodule`、`group`、`version`、`filter`等的配置信息，最后会将这些信息封装到 `com.weibo.api.motan.config.springsupport.ServiceConfigBean` 的对象中。

先来看一下 `ServiceConfigBean` 的UML图：

![Motan_ServiceConfigBean_UML](http://image.feathers.top/image/Motan_ServiceConfigBean_UML.jpg)

图中最左侧的继承关系是 `ServiceConfigBean` 自身的继承关系，右侧的是Spring的相关扩展。这个图这里先有个印象，我们继续上面的思路走。

Motan将配置信息封装到 `ServiceConfigBean` 后，调用了 `afterPropertiesSet()` 方法，由上图可知，这个方法是 `InitializingBean` 接口中抽象方法的实现。

这个方法其实就干了下面三件事儿：

```java
@Override
public void afterPropertiesSet() throws Exception {
    // 检查并配置basicConfig
    checkAndConfigBasicConfig();
    // 检查是否已经装配export，如果没有则到basicConfig查找
    checkAndConfigExport();
    // 检查并配置registry
    checkAndConfigRegistry();
}
```

* 检查配置解析过程后，ServiceConfigBean的 `basicService` 属性是否为空，如果是空，需要重新解析并设置他的值。

如何重新设置他的值呢？从UML图中我们可以找到 `basicService` 的位置，在 `ServiceConfig` 类中的第7个Field。字段类型是 `BasicServiceInterfaceConfig`。这个检查其实就是找到当前Spring容器中所有 `BasicServiceInterfaceConfig` 类型的bean，如果只找到一个，就把这个赋值到 `basicService` 上，如果有多个，需要找到 `BasicServiceInterfaceConfig` 的 `isDefault` 属性为true的那个，并赋值。

* 检查 `export` 的值是否已经设置，如果没有设置，到 `basicService` 中查找。

这一步其实是检查 `protocol`，也就是 `motan`、`motan2`这些协议是否已经设置好，export字段的格式为：`protocol1:port1,protocol2:port2`。对应到UML中，export字段在 `AbstractServiceConfig` 类中。
这里同时会将 `export` 的值解析到 `AbstractInterfaceConfig` 的 `protocols` 字段中。

* 检查注册中心的配置

例如 `zookeeper`、`consul` 这些是否已经配置好。如果是空的话，还是从 `basicService` 中查找，并将结果配置到 `AbstractInterfaceConfig` 的 `registries` 属性中。

这些都检查好以后，`basicService`、`export`、`protocols`、`registries`这些字段就初始化好了。然后会将新创建出来的这个 `ServiceConfigBean` 实例添加到 `AnnotationBean` 的 `serviceConfigs` 属性中。

```java
private final Set<ServiceConfigBean<?>> serviceConfigs = new ConcurrentHashSet<ServiceConfigBean<?>>();
```

至此 `@MotanService` 解析完成，可以准备发布并注册服务了。

### 2 服务的注册

完成上述的解析和初始化后，会调用 `ServiceConfigBean` 的 `export()` 方法来发布并注册服务。

```java
serviceConfig.export();
```

其实现如下：

```java
public synchronized void export() { // 这里加了个并发的控制，锁使用的是 this
    // 如果已经发布过了，直接返回
    if (exported.get()) {
        LoggerUtil.warn(String.format("%s has already been expoted, so ignore the export request!", interfaceClass.getName()));
        return;
    }

    // 检查暴露服务的类是否是某个接口的实现，如果不是则抛出异常
    // 检查暴露的方法是否在接口中存在，如果没有则抛出异常
    checkInterfaceAndMethods(interfaceClass, methods);

    List<URL> registryUrls = loadRegistryUrls();
    if (registryUrls == null || registryUrls.size() == 0) {
        throw new IllegalStateException("Should set registry config for service:" + interfaceClass.getName());
    }

    Map<String, Integer> protocolPorts = getProtocolAndPort();
    for (ProtocolConfig protocolConfig : protocols) {
        Integer port = protocolPorts.get(protocolConfig.getId());
        if (port == null) {
            throw new MotanServiceException(String.format("Unknow port in service:%s, protocol:%s", interfaceClass.getName(),
                    protocolConfig.getId()));
        }
        doExport(protocolConfig, port, registryUrls);
    }

    afterExport();
}
```