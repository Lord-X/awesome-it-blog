## Motan系列文章

* [Motan如何完成与Spring的集成](https://github.com/Lord-X/awesome-it-blog/blob/master/Motan/Motan_%E5%A6%82%E4%BD%95%E5%AE%8C%E6%88%90%E4%B8%8ESpring%E7%9A%84%E9%9B%86%E6%88%90.md)
* [Motan的SPI插件扩展机制](https://github.com/Lord-X/awesome-it-blog/blob/master/Motan/Motan_SPI%E6%8F%92%E4%BB%B6%E6%89%A9%E5%B1%95%E6%9C%BA%E5%88%B6.md)
* [Motan服务注册](https://github.com/Lord-X/awesome-it-blog/blob/master/Motan/Motan_%E6%9C%8D%E5%8A%A1%E6%B3%A8%E5%86%8C.md)
* [Motan服务调用](https://github.com/Lord-X/awesome-it-blog/blob/master/Motan/Motan_%E6%9C%8D%E5%8A%A1%E8%B0%83%E7%94%A8.md)

---

Motan是新浪微博研发并开源的一个RPC框架，与Dubbo相比，他更轻量级一些，代码也更少一些，但也五脏俱全。

Motan在GitHub上的项目地址：[https://github.com/weibocom/motan](https://github.com/weibocom/motan)

关于Motan的使用，可以看官方Wiki：[https://github.com/weibocom/motan/wiki/zh_overview](https://github.com/weibocom/motan/wiki/zh_overview)

基于Xml和基于Annotation的使用方式这里不再赘述，下面主要关注他是如何解析的。

### 0 如何扫描解析Xml文件

首先我们要知道的是，Spring是如何识别、解析Xml文件中那一大堆标签的。

这里给出一个Xml的示例：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
xmlns:motan="http://api.weibo.com/schema/motan"
xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-2.5.xsd
   http://api.weibo.com/schema/motan http://api.weibo.com/schema/motan.xsd">

    <!-- reference to the remote service -->
    <motan:referer id="remoteService" interface="quickstart.FooService" directUrl="localhost:8002"/>
</beans>
```

Xml文件通过xmlns定义命名空间，比如这里的 `<motan:referer ... >` 标签就是通过 `xmlns:motan="http://api.weibo.com/schema/motan"` 来定义的。

那么，写了这些东西后，Spring如何识别motan标签呢？

在Spring启动时会自动扫描 `classpath` 下的 `META-INF/spring.handlers` 文件，在motan中，此文件在motan-core下：

```text
motan-core/src/main/resources/META-INF/spring.handlers
```

此文件中定义了motan标签的解析器：

```text
http\://api.weibo.com/schema/motan=com.weibo.api.motan.config.springsupport.MotanNamespaceHandler
```

到这里可以发现，其实 `xmlns:motan` 的值，就是 `spring.handlers` 中配置的key，Spring就可以通过这个可以找到解析器 `MotanNamespaceHandler`。

#### 0.1 MotanNamespaceHandler

MotanNamespaceHandler在 `motan-springsupport` 工程下，它的实现如下：

```java
public class MotanNamespaceHandler extends NamespaceHandlerSupport {
    public final static Set<String> protocolDefineNames = new ConcurrentHashSet<String>();
    public final static Set<String> registryDefineNames = new ConcurrentHashSet<String>();
    public final static Set<String> basicServiceConfigDefineNames = new ConcurrentHashSet<String>();
    public final static Set<String> basicRefererConfigDefineNames = new ConcurrentHashSet<String>();

    @Override
    public void init() {
        registerBeanDefinitionParser("referer", new MotanBeanDefinitionParser(RefererConfigBean.class, false));
        registerBeanDefinitionParser("service", new MotanBeanDefinitionParser(ServiceConfigBean.class, true));
        registerBeanDefinitionParser("protocol", new MotanBeanDefinitionParser(ProtocolConfig.class, true));
        registerBeanDefinitionParser("registry", new MotanBeanDefinitionParser(RegistryConfig.class, true));
        registerBeanDefinitionParser("basicService", new MotanBeanDefinitionParser(BasicServiceInterfaceConfig.class, true));
        registerBeanDefinitionParser("basicReferer", new MotanBeanDefinitionParser(BasicRefererInterfaceConfig.class, true));
        registerBeanDefinitionParser("spi", new MotanBeanDefinitionParser(SpiConfigBean.class, true));
        registerBeanDefinitionParser("annotation", new MotanBeanDefinitionParser(AnnotationBean.class, true));
        Initializable initialization = InitializationFactory.getInitialization();
        // 这里用于初始化SPI扩展点，本文暂不介绍。
        initialization.init();
    }
}
```

它继承于Spring中的 `NamespaceHandlerSupport`，并实现 `init()` 了方法，此方法用于初始化标签解析器。

调用 `NamespaceHandlerSupport` 的 `registerBeanDefinitionParser` 方法，可完成解析器的注册，motan统一用 `MotanBeanDefinitionParser` 作为解析器，通过不同的参数来处理不同的标签。
`MotanBeanDefinitionParser` 继承于Spring的 `BeanDefinitionParser`，并实现了其 `parse()` 方法，这个方法就会在解析Xml文件时，将遇到的element解析并生成BeanDefinition，并将其注册到上下文的 `BeanDefinitionRegistry` 中，最终完成Xml的解析。


### 1 如何扫描解析Annotation

有两种方式开启Annotation模式:
* Xml中配置
* @Bean配置

使用这两种方式都会生成 `AnnotationBean`，并将其注入到Spring容器中。 

#### 1.1 Xml配置形式

此方式依然是通过 `META-INF/spring.handlers` 完成的。

由于容器启动需要扫描此文件，在上文描述中，指定的 `MotanNamespaceHandler` 中的 `init()` 方法，通过以下代码提供了annotation这个element的解析：

```java
@Override
public void init() {
    // ...
    registerBeanDefinitionParser("annotation", new MotanBeanDefinitionParser(AnnotationBean.class, true));
    // ... 
}
```

然后在Xml配置中配置：

```xml
<motan:annotation package="xxx.xxx.xx"/>
```

就可以扫描指定package下面的类了。

#### 1.2 @Bean形式

通过以下方式扫描package指定的类：

```java
@Bean
public AnnotationBean motanAnnotationBean() {
    AnnotationBean motanAnnotationBean = new AnnotationBean();
    motanAnnotationBean.setPackage("xxx.xxx.xx");
    return motanAnnotationBean;
}
```

#### 1.3 motan核心注解

motan有两个核心注解：
* com.weibo.api.motan.config.springsupport.annotation.MotanReferer
* com.weibo.api.motan.config.springsupport.annotation.MotanService

其中，`@MotanService` 用于标识某个类是一个motan服务实现类，`MotanReferer` 用于标识服务调用方。

典型的使用方式如下：

* 定义一个接口

```java
public interface MotanDemoService {
    String hello(name);
}
```

* 提供其实现类，并暴露服务

```java
@MotanService(/* protocol、registry 等配置 */)
public class MotanDemoServiceImpl implements MotanDemoService {

    public String hello(String name) {
        System.out.println(name);
        return "Hello " + name + "!";
    }
}
```

* 服务调用

```java
@MotanReferer(/* 配置 */)
private MotanDemoService service;
```

#### 1.4 核心注解的解析

将 `AnnotationBean` 注入到容器时，motan通过Spring提供的功能介入了Bean的生命周期，来完成注解的扫描。

来看一下 `AnnotationBean` 的定义：

```java
public class AnnotationBean implements DisposableBean, BeanFactoryPostProcessor, BeanPostProcessor, BeanFactoryAware, Ordered {
    // ...
}
```

最关键的在于 `BeanFactoryPostProcessor` 和 `BeanPostProcessor`。对他俩做个简单介绍：
* BeanFactoryPostProcessor：允许我们在容器实例化相应对象之前，对注册到容器的BeanDefinition所保存的信息做响应的修改。
* BeanPostProcessor：允许在Spring Bean对象的初始化方法（init-method）的调用前后来接入初始化阶段。

引用一个网络上的图来描述一下这两个接口介入的时机：

![Spring-Bean生命周期](http://image.feathers.top/image/Spring-Bean生命周期.png)

按照顺序，首先看一下 `BeanFactoryPostProcessor` 的 `postProcessBeanFactory` 都干了啥。

```java
@Override
public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory)
        throws BeansException {
    if (annotationPackage == null || annotationPackage.length() == 0) {
        return;
    }
    if (beanFactory instanceof BeanDefinitionRegistry) {
        try {
            // 初始化类扫描器
            Class<?> scannerClass = ClassUtils.forName("org.springframework.context.annotation.ClassPathBeanDefinitionScanner",
                    AnnotationBean.class.getClassLoader());
            // 实例化类扫描器
            Object scanner = scannerClass.getConstructor(new Class<?>[]{BeanDefinitionRegistry.class, boolean.class})
                    .newInstance(new Object[]{(BeanDefinitionRegistry) beanFactory, true});
            // 初始化注解类型过滤器，在包扫描的过程中，满足指定条件注解的class会被加载
            Class<?> filterClass = ClassUtils.forName("org.springframework.core.type.filter.AnnotationTypeFilter",
                    AnnotationBean.class.getClassLoader());
            // 设置要匹配的注解：@MotanService，即暴露motan服务的类
            Object filter = filterClass.getConstructor(Class.class).newInstance(MotanService.class);
            Method addIncludeFilter = scannerClass.getMethod("addIncludeFilter",
                    ClassUtils.forName("org.springframework.core.type.filter.TypeFilter", AnnotationBean.class.getClassLoader()));
            // 设置"包含"过滤器，只有包含上面指定的@MotanService注解时，才加载
            addIncludeFilter.invoke(scanner, filter);
            // 执行扫描
            Method scan = scannerClass.getMethod("scan", new Class<?>[]{String[].class});
            scan.invoke(scanner, new Object[]{annotationPackages});
        } catch (Throwable e) {
            // spring 2.0
        }
    }
}
```

变量 `annotationPackage` 即上面设置的要扫描的包。代码大意已在注释中给出。这段代码的作用，就是扫描指定的包，并将符合条件的class作为Spring候选Bean，并将其BeanDefinition注册到给定的BeanFactory或ApplicationContext中。

然后来看下 `BeanPostProcessor` 的两个方法干了啥。

`BeanPostProcessor` 有两个方法：`postProcessBeforeInitialization` 和 `postProcessAfterInitialization`。

`postProcessBeforeInitialization` 用于 `init-mothod` 方法执行前介入，`postProcessAfterInitialization` 在其后介入。两个方法的实现如下：

* postProcessBeforeInitialization

```java
/**
 * init reference field
 */
@Override
public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
    if (!isMatchPackage(bean)) {
        return bean;
    }
    Class<?> clazz = bean.getClass();
    if (isProxyBean(bean)) {
        clazz = AopUtils.getTargetClass(bean);
    }
    // ... 解析setter method上的@MotanReferer注解，省略 ...

    Field[] fields = clazz.getDeclaredFields();
    for (Field field : fields) {
        try {
            if (!field.isAccessible()) {
                field.setAccessible(true);
            }
            // 获取field上的MotanReferer注解，如果获取到了，通过refer方法解析他
            MotanReferer reference = field.getAnnotation(MotanReferer.class);
            if (reference != null) {
                Object value = refer(reference, field.getType());
                if (value != null) {
                    field.set(bean, value);
                }
            }
        } catch (Throwable t) {
            throw new BeanInitializationException("Failed to init remote service reference at filed " + field.getName()
                    + " in class " + bean.getClass().getName(), t);
        }
    }
    return bean;
}
```

* postProcessAfterInitialization

```java
/**
 * init service config and export servcice
 */
@Override
public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
    if (!isMatchPackage(bean)) {
        return bean;
    }
    Class<?> clazz = bean.getClass();
    if (isProxyBean(bean)) {
        clazz = AopUtils.getTargetClass(bean);
    }
    // 获取class上的@MotanService注解，并解析
    MotanService service = clazz.getAnnotation(MotanService.class);
    if (service != null) {
        ServiceConfigBean<Object> serviceConfig = new ServiceConfigBean<Object>();
        // ... 初始化ServiceConfigBean，省略 ...
    }
    return bean;
}
```

`postProcessBeforeInitialization` 方法用于解析Bean中带有@MotanReferer注解的setter方法或field，并完成调用方的初始化。

`postProcessAfterInitialization` 方法用于解析带有@MotanService注解的class，并将这个class作为motan服务注册到注册中心，暴露为服务。


### 2 总结

motan也有原生的服务暴露形式，本文没有介绍，具体可以参考官方wiki。

motan主要利用Spring启动时加载并初始化 `META-INF/spring.handlers` 来完成与Spring的集成。

在xml配置形式下，motan用 `MotanNamespaceHandler` 完成标签解析器的注册。

在Annotation配置形式下，motan主要利用 `BeanFactoryPostProcessor` 和 `BeanPostProcessor` 介入Bean生命周期，用 `BeanFactoryPostProcessor` 实现了class的扫描，用 `BeanPostProcessor` 实现了两个核心注解 `@MotanReferer` 和 `@MotanService` 的解析。


