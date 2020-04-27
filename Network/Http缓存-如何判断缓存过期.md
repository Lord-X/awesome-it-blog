## Http缓存：如何判断缓存是否过期

缓存是否过期主要与Response Header的两类头部有关，一个是缓存最大有效时间（记为freshness_lifetime），另一个是缓存已经存在的时间（记为current_age）。

有个上面两个值后，就可以判断缓存是否过期了，逻辑如下：

```java
if (freshness_lifetime > current_age) {
    未过期
} else {
    已过期
}
```

接下来分别看一下这两个值怎么计算。

### freshness_lifetime的计算

freshness_lifetime的值取自Response Header，优先级如下：

```text
s-maxage > max-age > Expires > 预估的过期时间(下文中解释)
```

下面针对前三种情况，举两个例子。

* 栗子1：s-maxage和max-age

![](http://image.feathers.top/image/HttpCache4.png)

如上图，某一资源这两个值同时出现，则 `freshness_lifetime` 的值取 `s-maxage` 。

* 栗子2：max-age和Expires

![](http://image.feathers.top/image/HttpCache5.png)

如上图，某一资源这两个值同时出现，则 `freshness_lifetime` 的值取 `max-age` 。

#### 预估过期时间

接下来说说 `预估的过期时间`。由于网络中某些资源没有通过max-age等头部告诉浏览器缓存这个资源（可能是服务器配置有问题），但这些资源通常是一些不长变化的静态资源，例如js、css等，这种情况下，浏览器为了性能考虑还是决定把它缓存。那缓存多久呢？现代浏览器通常是基于RFC7234推荐的计算方法，即：

```text
(DownloadTime - LastModified) * 10%
```

* DownloadTime：浏览器获取到响应的时间
* LastModified：服务端资源上次修改时间

这个值的优先级是最低的，只有当Response Header没有返回前三个头部，并且浏览器决定缓存这个资源的时候，才会使用他。

### current_age的计算

current_age的计算涉及到Response Header的 `age` 头部，所以我们先来明确一下 `age` 的含义。

`age` 表示自源服务器发出资源的响应，到客户端使用这个资源的缓存时，经过的秒数。

这里一定要注意的是：自源服务器发出响应的时间。举个例子：

一个资源从源服务器发出后，可能会经过多层代理服务器，最后到达客户端，而每一层代理服务器也有可能把这个资源缓存，所以age的计算是要加上每一层代理服务器的缓存时间的，例如下图：

![](http://image.feathers.top/image/20200427115940.png)

由上图可知，虽然Browser是直接从 `代理服务器1` 获取到的 `aaa.js`，但是 `aaa.js` 从源服务器发出的时间是 12:00:00，所以 `age` 应该是 Browser 接收到响应的时间（12:02:10）减去源服务器发出的时间（12:00:00），等于130秒。

**PS：上例中的age计算过程是粗略的计算，实际计算时，还要计算每一个代理间的响应时延。**

以上就是 `current_age` 的计算方式。

## 参考

极客时间 - Web协议详解与抓包实战