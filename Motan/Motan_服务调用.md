## Motan系列文章

* [Motan如何完成与Spring的集成](https://github.com/Lord-X/awesome-it-blog/blob/master/Motan/Motan_%E5%A6%82%E4%BD%95%E5%AE%8C%E6%88%90%E4%B8%8ESpring%E7%9A%84%E9%9B%86%E6%88%90.md)
* [Motan的SPI插件扩展机制](https://github.com/Lord-X/awesome-it-blog/blob/master/Motan/Motan_SPI%E6%8F%92%E4%BB%B6%E6%89%A9%E5%B1%95%E6%9C%BA%E5%88%B6.md)
* [Motan服务注册](https://github.com/Lord-X/awesome-it-blog/blob/master/Motan/Motan_%E6%9C%8D%E5%8A%A1%E6%B3%A8%E5%86%8C.md)
* [Motan服务调用](https://github.com/Lord-X/awesome-it-blog/blob/master/Motan/Motan_%E6%9C%8D%E5%8A%A1%E8%B0%83%E7%94%A8.md)

---

### 0 @MotanReferer注解是个啥

被 `@MotanReferer` 标注的 setter 方法或 field 会被motan在启动时扫描，并为其创建动态代理，并将动态代理的实例赋值给这个 field。远程服务的调用都是在这个代理中实现的。

下面以注解在 field 的情况为例，说明 `@MotanReferer` 的解析以及创建动态代理的过程：

```java
@MotanReferer(basicReferer = "ad-commonBasicRefererConfigBean", application = "ad-filter", version = "1.1.0")
private AdCommonRPC adCommonRPC;
```

关于如何扫描，在 [Motan如何完成与Spring的集成](https://github.com/Lord-X/awesome-it-blog/blob/master/Motan/Motan_%E5%A6%82%E4%BD%95%E5%AE%8C%E6%88%90%E4%B8%8ESpring%E7%9A%84%E9%9B%86%E6%88%90.md) 一文中有详细说明，这里不再赘述。

### 1 @MotanReferer注解的解析

这个过程始于 `AnnotationBean` 中对扫描到的bean的field的解析。Motan会解析出带有 `@MotanReferer` 的field，应调用 AnnotationBean 的 `refer` 方法初始化并创建代理。field的解析如下：

```java
Field[] fields = clazz.getDeclaredFields();
for (Field field : fields) {
    try {
        if (!field.isAccessible()) {
            field.setAccessible(true);
        }
        MotanReferer reference = field.getAnnotation(MotanReferer.class);
        if (reference != null) {
            // 调用 refer 方法初始化并创建动态代理
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
```

### 2 @MotanReferer的初始化


