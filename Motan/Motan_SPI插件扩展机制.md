## Motan系列文章

* [Motan如何完成与Spring的集成](https://github.com/Lord-X/awesome-it-blog/blob/master/Motan/Motan_%E5%A6%82%E4%BD%95%E5%AE%8C%E6%88%90%E4%B8%8ESpring%E7%9A%84%E9%9B%86%E6%88%90.md)
* [Motan的SPI插件扩展机制](https://github.com/Lord-X/awesome-it-blog/blob/master/Motan/Motan_SPI%E6%8F%92%E4%BB%B6%E6%89%A9%E5%B1%95%E6%9C%BA%E5%88%B6.md)
* [Motan服务注册](https://github.com/Lord-X/awesome-it-blog/blob/master/Motan/Motan_%E6%9C%8D%E5%8A%A1%E6%B3%A8%E5%86%8C.md)
* [Motan服务调用](https://github.com/Lord-X/awesome-it-blog/blob/master/Motan/Motan_%E6%9C%8D%E5%8A%A1%E8%B0%83%E7%94%A8.md)

---

### 0 Motan中的SPI

Motan的SPI与Dubbo的SPI类似，它在Java原生SPI思想的基础上做了优化，并且与Java原生SPI的使用方式很相似。

介绍Java原生SPI的相关文章有很多，这里不再赘述。下面主要介绍一下Motan中的SPI机制，先从使用说起。

#### 0.1 SPI的使用

下面以实现一个Filter的扩展为例，说说他咋用。

Filter的作用是在 `Provider` 端接收请求或 `Consumer` 端发送请求时进行拦截，做一些特别的事情。下面从 `Consumer` 端的视角，做一个计算单次请求响应时间的统计需求。

首先需要写一个类，实现 `com.weibo.api.motan.filter.Filter` 接口的 `filter` 方法。

```java
@SpiMeta(name = "profiler")
public class ProfilerFilter implements Filter {
    
    @Override
    public Response filter(Caller<?> caller, Request request) {
        // 记录开始时间
        long begin = System.nanoTime();
        try {
            final Response response = caller.call(request);
            return response;
        } finally {
            // 打印本次响应时间
            System.out.println("Time cost : " + (System.nanoTime() - begin));
        }
    }
}
```

其次，在 `META-INF/services` 目录下创建名为 `com.weibo.api.motan.filter.Filter` 的文件，内容如下：

```text
# 例如：com.pepper.metrics.integration.motan.MotanProfilerFilter
#全限定名称#.MotanProfilerFilter
```

然后给Protocol配置 `filter` 

```java
ProtocolConfigBean config = new ProtocolConfigBean();
config.setName("motan");
config.setMaxContentLength(1048576);
config.setFilter("profiler"); // 配置filter
return config;
```

最后在 `RefererConfig` 中使用这个 `ProtocolConfig` 即可。

```java
BasicRefererConfigBean config = new BasicRefererConfigBean();
config.setProtocol("demoMotan");
// ... 省略其他配置 ...

```

如此一来，在 `Consumer` 端就可以拦截每次请求，并打印响应时间了。

接下来，继续研究一下Motan是如何做到这件事的。

### 1 SPI的管理

Motan的SPI的实现在 `motan-core/com/weibo/api/motan/core/extension` 中。组织结构如下：

```text
motan-core/com.weibo.api.motan.core.extension
    |-Activation：SPI的扩展功能，例如过滤、排序
    |-ActivationComparator：排序比较器
    |-ExtensionLoader：核心，主要负责SPI的扫描和加载
    |-Scope：模式枚举，单例、多例
    |-Spi：注解，作用在接口上，表明这个接口的实现可以通过SPI的形式加载
    |-SpiMeta：注解，作用在具体的SPI接口的实现类上，标注该扩展的名称

```

#### 1.1 内部管理的数据结构

```java
private static ConcurrentMap<Class<?>, ExtensionLoader<?>> extensionLoaders = new ConcurrentHashMap<Class<?>, ExtensionLoader<?>>();
private ConcurrentMap<String, T> singletonInstances = null;
private ConcurrentMap<String, Class<T>> extensionClasses = null;
private Class<T> type;
private ClassLoader classLoader; // 类加载使用的ClassLoader
```

`extensionLoaders` 是类变量，他管理的是由 `@Spi` 注解标注的接口与其 `ExtensionLoader` 的映射，作为所有SPI的全局管理器。

`singletonInstances` 维护了当前 `ExtensionLoader` 中的单例扩展。

`extensionClasses` 维护了当前 `ExtensionLoader` 所有扩展实例的Class对象，用于创建多例（通过class.newInstance创建）。

`type` 维护了当前 `@Spi` 注解标注的接口的 `class` 对象。

#### 1.2 ExtensionLoader的初始化

在 `Motan` 中，可以通过以下方式初始化ExtensionLoader（以上文中的Filter SPI为例）：

```java
// 初始化 Filter 到全局管理器 `extensionLoaders` 中
ExtensionLoader extensionLoader = ExtensionLoader.getExtensionLoader(Filter.class);

```

然后我们具体看一下 `ExtensionLoader.getExtensionLoader(Filter.class)` 都干了啥。

```java
public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
    // 检查type是否为空、type是否是接口类型、type是否被@Spi标注，检查失败会抛出异常
    checkInterfaceType(type);
    // 尝试从上文提到的 `extensionLoaders` 管理器中获取已有的ExtensionLoader
    ExtensionLoader<T> loader = (ExtensionLoader<T>) extensionLoaders.get(type);

    // 获取失败的话，尝试扫描并加载指定type的扩展，并初始化之
    if (loader == null) {
        loader = initExtensionLoader(type);
    }
    return loader;
}
```

然后看看 `initExtensionLoader` 方法干了啥。

```java
// synchronized锁控制，防止并发初始化
public static synchronized <T> ExtensionLoader<T> initExtensionLoader(Class<T> type) {
    ExtensionLoader<T> loader = (ExtensionLoader<T>) extensionLoaders.get(type);
    // 二层检查，防止并发问题
    if (loader == null) {
        // type会赋值给实例变量 `type`，并初始化实例变量 `classLoader` 为当前线程上线文的ClassLoader
        loader = new ExtensionLoader<T>(type);
        // 添加到全局管理器 `extensionLoaders` 中
        extensionLoaders.putIfAbsent(type, loader);

        loader = (ExtensionLoader<T>) extensionLoaders.get(type);
    }

    return loader;
}
```

至此，我们就初始化了 `Filter` 接口的 `ExtensionLoader`，并将它托管到了 `extensionLoaders` 中。

#### 1.3 SPI的扫描和加载以及获取指定的扩展实例

`Motan` 是懒加载策略，当第一次获取具体的某一扩展实例时，才会扫描和加载所有的扩展实例。

例如，可以通过以下方式获取我们上面创建的名为 `profiler` 的 `Filter` 接口的扩展实例。

```java
// 初始化 Filter 到全局管理器 `extensionLoaders` 中
ExtensionLoader extensionLoader = ExtensionLoader.getExtensionLoader(Filter.class);
Filter profilterFilter = extensionLoader.getExtension("profiler");
```

`Filter` 接口的 `ExtensionLoader` 实际上是在第一次 `extensionLoader.getExtension("profiler")` 时完成的。下面具体看一下 `getExtension` 方法干了啥。

```java
public T getExtension(String name) {
    // Notes：就是通过这个checkInit方法扫描和加载的
    checkInit();
    // .. 暂时省略 ..
}
```

继续研究 `checkInit` 方法。

```java
private volatile boolean init = false;
private void checkInit() {
    // 用init标记，只初始化一次
    if (!init) {
        loadExtensionClasses();
    }
}

private static final String PREFIX = "META-INF/services/";
private synchronized void loadExtensionClasses() {
    if (init) {
        return;
    }
    // 扫描和加载
    extensionClasses = loadExtensionClasses(PREFIX);
    singletonInstances = new ConcurrentHashMap<String, T>();

    init = true;
}
```

`loadExtensionClasses` 方法会扫描 `META-INF/services/` 下的所有文件，并解析文件内容，它会调用 `loadClass` 方法，该方法实现如下：

```java
// classNames 就是每个文件中的具体扩展实现的全限定名称。
private ConcurrentMap<String, Class<T>> loadClass(List<String> classNames) {
    ConcurrentMap<String, Class<T>> map = new ConcurrentHashMap<String, Class<T>>();

    for (String className : classNames) {
        try {
            Class<T> clz;
            if (classLoader == null) {
                clz = (Class<T>) Class.forName(className);
            } else {
                clz = (Class<T>) Class.forName(className, true, classLoader);
            }

            checkExtensionType(clz);
            // 获取 @SpiMeta 注解声明的名称
            String spiName = getSpiName(clz);

            if (map.containsKey(spiName)) {
                failThrows(clz, ":Error spiName already exist " + spiName);
            } else {
                map.put(spiName, clz);
            }
        } catch (Exception e) {
            failLog(type, "Error load spi class", e);
        }
    }

    return map;
}
```

这个方法做的事情就是获取所有合法的扩展的class。最终其返回值会赋值给实例变量 `extensionClasses`，至此完成了扫描和加载工作。

> PS：由上可知，extensionClasses的K-V是具体扩展实现的 @SpiMeta 名称和对应class的映射。以上文的 `ProfilerFilter` 为例来说，KEY=profiler，VALUE=ProfilerFilter.class

#### 1.4 获取具体的SPI扩展实现

继续看刚才 `getExtension` 方法中省略的部分。

```java
public T getExtension(String name) {
    checkInit();

    if (name == null) {
        return null;
    }

    try {
        Spi spi = type.getAnnotation(Spi.class);

        // 获取单例
        if (spi.scope() == Scope.SINGLETON) {
            return getSingletonInstance(name);
        } else {
            // 获取多例
            Class<T> clz = extensionClasses.get(name);

            if (clz == null) {
                return null;
            }

            return clz.newInstance();
        }
    } catch (Exception e) {
        failThrows(type, "Error when getExtension " + name, e);
    }

    return null;
}
```

获取多例的情况很容易，直接从前面加载好的 `extensionClasses` 中获取，如果获取到就 `newInstance()` 一个新的实例。

下面看下单例的情况：`getSingletonInstance` 方法。

```java
private T getSingletonInstance(String name) throws InstantiationException, IllegalAccessException {
    T obj = singletonInstances.get(name);

    if (obj != null) {
        return obj;
    }

    Class<T> clz = extensionClasses.get(name);

    if (clz == null) {
        return null;
    }

    synchronized (singletonInstances) {
        obj = singletonInstances.get(name);
        if (obj != null) {
            return obj;
        }

        obj = clz.newInstance();
        singletonInstances.put(name, obj);
    }

    return obj;
}
```

获取单例时，会优先尝试从单例集合 `singletonInstances` 中获取，如果获取不到，说明这个单例实例还没添加到单例集合中（或没有对应名称的具体实例），然后会尝试从 `extensionClasses` 中获取，如果还获取不到，就是真没有了，如果获取到，会new一个instance到单例集合中。

以上就是 Motan 获取具体SPI扩展实现的方式。

以上便是 Motan 的 SPI 机制。