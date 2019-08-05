## 个人原创技术文章

---



> 欢迎关注个人微信公众号【Mr羽墨青衫】，目前主要关注Java本身和性能优化层面。
>
> ![wx](http://feathers.zrbcool.top/image/%E5%85%AC%E4%BC%97%E5%8F%B7%E4%BA%8C%E7%BB%B4%E7%A0%81.jpg)

### Index
* [Java](https://github.com/Lord-X/awesome-it-blog/tree/master/java)
   * [HashMap原理及内部存储结构](https://github.com/Lord-X/awesome-it-blog/blob/master/java/HashMap%E5%8E%9F%E7%90%86%E5%8F%8A%E5%86%85%E9%83%A8%E5%AD%98%E5%82%A8%E7%BB%93%E6%9E%84.md)
   * [JVM-GC-串行回收器-SerialGC实战](https://github.com/Lord-X/awesome-it-blog/blob/master/java/JVM-GC-%E4%B8%B2%E8%A1%8C%E5%9B%9E%E6%94%B6%E5%99%A8-SerialGC%E5%AE%9E%E6%88%98.md)
   * [JVM-GC垃圾回收算法-判定一个对象是否是可回收的对象](https://github.com/Lord-X/awesome-it-blog/blob/master/java/JVM-GC%E5%9E%83%E5%9C%BE%E5%9B%9E%E6%94%B6%E7%AE%97%E6%B3%95-%E5%88%A4%E5%AE%9A%E4%B8%80%E4%B8%AA%E5%AF%B9%E8%B1%A1%E6%98%AF%E5%90%A6%E6%98%AF%E5%8F%AF%E5%9B%9E%E6%94%B6%E7%9A%84%E5%AF%B9%E8%B1%A1.md)
   * [JVM-GC垃圾回收算法-引用计数法](https://github.com/Lord-X/awesome-it-blog/blob/master/java/JVM-GC%E5%9E%83%E5%9C%BE%E5%9B%9E%E6%94%B6%E7%AE%97%E6%B3%95-%E5%BC%95%E7%94%A8%E8%AE%A1%E6%95%B0%E6%B3%95.md)
   * [JVM-GC垃圾回收算法-标记清除法、复制算法、标记压缩法、分代算法](https://github.com/Lord-X/awesome-it-blog/blob/master/java/JVM-GC%E5%9E%83%E5%9C%BE%E5%9B%9E%E6%94%B6%E7%AE%97%E6%B3%95-%E6%A0%87%E8%AE%B0%E6%B8%85%E9%99%A4%E6%B3%95%E3%80%81%E5%A4%8D%E5%88%B6%E7%AE%97%E6%B3%95%E3%80%81%E6%A0%87%E8%AE%B0%E5%8E%8B%E7%BC%A9%E6%B3%95%E3%80%81%E5%88%86%E4%BB%A3%E7%AE%97%E6%B3%95.md)
   * [Java8中ConcurrentHashMap是如何保证线程安全的](https://github.com/Lord-X/awesome-it-blog/blob/master/java/Java8%E4%B8%ADConcurrentHashMap%E6%98%AF%E5%A6%82%E4%BD%95%E4%BF%9D%E8%AF%81%E7%BA%BF%E7%A8%8B%E5%AE%89%E5%85%A8%E7%9A%84.md)
   * [Java8迁移到Java11的过程](https://github.com/Lord-X/awesome-it-blog/blob/master/java/Java8%E8%BF%81%E7%A7%BB%E5%88%B0Java11%E7%9A%84%E8%BF%87%E7%A8%8B.md)
   * [Java虚拟机-即时编译器（上）](https://github.com/Lord-X/awesome-it-blog/blob/master/java/Java%E8%99%9A%E6%8B%9F%E6%9C%BA-%E5%8D%B3%E6%97%B6%E7%BC%96%E8%AF%91%E5%99%A8%EF%BC%88%E4%B8%8A%EF%BC%89.md)
   * [Java虚拟机-即时编译器（下）](https://github.com/Lord-X/awesome-it-blog/blob/master/java/Java%E8%99%9A%E6%8B%9F%E6%9C%BA-%E5%8D%B3%E6%97%B6%E7%BC%96%E8%AF%91%E5%99%A8%EF%BC%88%E4%B8%8B%EF%BC%89.md)
   * [Java队列同步器（AQS）到底是怎么一回事](https://github.com/Lord-X/awesome-it-blog/blob/master/java/Java%E9%98%9F%E5%88%97%E5%90%8C%E6%AD%A5%E5%99%A8%EF%BC%88AQS%EF%BC%89%E5%88%B0%E5%BA%95%E6%98%AF%E6%80%8E%E4%B9%88%E4%B8%80%E5%9B%9E%E4%BA%8B.md)
   * [Unsafe类的介绍和使用](https://github.com/Lord-X/awesome-it-blog/blob/master/java/Unsafe%E7%B1%BB%E7%9A%84%E4%BB%8B%E7%BB%8D%E5%92%8C%E4%BD%BF%E7%94%A8.md)
   * [深入分析Java类加载器原理](https://github.com/Lord-X/awesome-it-blog/blob/master/java/%E6%B7%B1%E5%85%A5%E5%88%86%E6%9E%90Java%E7%B1%BB%E5%8A%A0%E8%BD%BD%E5%99%A8%E5%8E%9F%E7%90%86.md)
   * [深入剖析LongAdder是咋干活的](https://github.com/Lord-X/awesome-it-blog/blob/master/java/%E6%B7%B1%E5%85%A5%E5%89%96%E6%9E%90LongAdder%E6%98%AF%E5%92%8B%E5%B9%B2%E6%B4%BB%E7%9A%84.md)
   * [深入剖析ReentrantLock原理](https://github.com/Lord-X/awesome-it-blog/blob/master/java/%E6%B7%B1%E5%85%A5%E5%89%96%E6%9E%90ReentrantLock%E5%8E%9F%E7%90%86.md)
   * [深入剖析线程同步工具CountDownLatch原理](https://github.com/Lord-X/awesome-it-blog/blob/master/java/%E6%B7%B1%E5%85%A5%E5%89%96%E6%9E%90%E7%BA%BF%E7%A8%8B%E5%90%8C%E6%AD%A5%E5%B7%A5%E5%85%B7CountDownLatch%E5%8E%9F%E7%90%86.md)
   * [深入解读HashMap线程安全性问题](https://github.com/Lord-X/awesome-it-blog/blob/master/java/%E6%B7%B1%E5%85%A5%E8%A7%A3%E8%AF%BBHashMap%E7%BA%BF%E7%A8%8B%E5%AE%89%E5%85%A8%E6%80%A7%E9%97%AE%E9%A2%98.md)
   * [关于Ahead-of-Time Compilation的调研与实践](https://github.com/Lord-X/awesome-it-blog/blob/master/java/%E5%85%B3%E4%BA%8EAhead-of-Time%20Compilation%E7%9A%84%E8%B0%83%E7%A0%94%E4%B8%8E%E5%AE%9E%E8%B7%B5.md)
   * [Java Flight Recorder初探](https://github.com/Lord-X/awesome-it-blog/blob/master/java/Java%20Flight%20Recorder%E5%88%9D%E6%8E%A2.md)
   * [关于Graal编译器的调研与实践](https://github.com/Lord-X/awesome-it-blog/blob/master/java/%E5%85%B3%E4%BA%8EGraal%E7%BC%96%E8%AF%91%E5%99%A8%E7%9A%84%E8%B0%83%E7%A0%94%E4%B8%8E%E5%AE%9E%E8%B7%B5.md)
   * [阿里JDK：Dragonwell8中JWarmup的调研与实践](https://github.com/Lord-X/awesome-it-blog/blob/master/java/%E9%98%BF%E9%87%8CJDK%EF%BC%9ADragonwell8%E4%B8%ADJWarmup%E7%9A%84%E8%B0%83%E7%A0%94%E4%B8%8E%E5%AE%9E%E8%B7%B5.md)
   
* [Database](https://github.com/Lord-X/awesome-it-blog/tree/master/DB)
   * [详细解读阿里手册之MySQL](https://github.com/Lord-X/awesome-it-blog/blob/master/DB/%E8%AF%A6%E7%BB%86%E8%A7%A3%E8%AF%BB%E9%98%BF%E9%87%8C%E6%89%8B%E5%86%8C%E4%B9%8BMySQL.md)

* [NoSQL](https://github.com/Lord-X/awesome-it-blog/tree/master/NoSQL)
   * [Redis-bgsave导致的接口响应延迟波动（深入分析Linux的fork()机制）](https://github.com/Lord-X/awesome-it-blog/blob/master/NoSQL/Redis-bgsave%E5%AF%BC%E8%87%B4%E7%9A%84%E6%8E%A5%E5%8F%A3%E5%93%8D%E5%BA%94%E5%BB%B6%E8%BF%9F%E6%B3%A2%E5%8A%A8%EF%BC%88%E6%B7%B1%E5%85%A5%E5%88%86%E6%9E%90Linux%E7%9A%84fork()%E6%9C%BA%E5%88%B6%EF%BC%89.md)
   * [Redis-探究-集群扩容导致的Jedis客户端报JedisMovedDataException异常](https://github.com/Lord-X/awesome-it-blog/blob/master/NoSQL/Redis-%E6%8E%A2%E7%A9%B6-%E9%9B%86%E7%BE%A4%E6%89%A9%E5%AE%B9%E5%AF%BC%E8%87%B4%E7%9A%84Jedis%E5%AE%A2%E6%88%B7%E7%AB%AF%E6%8A%A5JedisMovedDataException%E5%BC%82%E5%B8%B8.md)

* [kubernetes](https://github.com/Lord-X/awesome-it-blog/tree/master/k8s)
   * [酷划商业平台容器化历程](https://github.com/Lord-X/awesome-it-blog/blob/master/k8s/%E9%85%B7%E5%88%92%E5%95%86%E4%B8%9A%E5%B9%B3%E5%8F%B0%E5%AE%B9%E5%99%A8%E5%8C%96%E5%8E%86%E7%A8%8B.md)
   * [附_酷划商业平台容器化后性能对比](https://github.com/Lord-X/awesome-it-blog/blob/master/k8s/%E9%99%84_%E9%85%B7%E5%88%92%E5%95%86%E4%B8%9A%E5%B9%B3%E5%8F%B0%E5%AE%B9%E5%99%A8%E5%8C%96%E5%90%8E%E6%80%A7%E8%83%BD%E5%AF%B9%E6%AF%94.md)