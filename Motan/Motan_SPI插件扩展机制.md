## Motan系列文章

* [Motan如何完成与Spring的集成](https://github.com/Lord-X/awesome-it-blog/blob/master/Motan/Motan_%E5%A6%82%E4%BD%95%E5%AE%8C%E6%88%90%E4%B8%8ESpring%E7%9A%84%E9%9B%86%E6%88%90.md)

---

### 0 Motan中的SPI

Motan的SPI与Dubbo的SPI类似，它在Java原生SPI思想的基础上做了优化，并且与Java原生SPI的使用方式很相似。

介绍Java原生SPI的相关文章有很多，这里不再赘述。下面主要介绍一下Motan中的SPI机制。

Motan的SPI的实现在 `motan-core/com/weibo/api/motan/core/extension` 中。