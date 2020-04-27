## Http缓存：论某一资源被缓存和使用缓存的条件

某一请求使用缓存的基本步骤有两个，一个是先将响应的资源缓存下来，另一个是在发起请求时满足使用缓存的条件。下面先来看看什么样的响应会被缓存下来。

### 什么样的响应会被缓存

这里引用RFC7234原文做简要说明（原文：page6，第三章节）

A cache MUST NOT store a response to any request, unless:
* The request method is understood by the cache and defined as being
cacheable, and
* the response status code is understood by the cache, and
* the "no-store" cache directive (see Section 5.2) does not appear
in request or response header fields, and
* the "private" response directive (see Section 5.2.2.6) does not
appear in the response, if the cache is shared, and
* the Authorization header field (see Section 4.2 of \[RFC7235\]) does
not appear in the request, if the cache is shared, unless the
response explicitly allows it (see Section 3.2), and
* the response either:
	- contains an Expires header field (see Section 5.3), or
	- contains a max-age response directive (see Section 5.2.2.8), or
	- contains a s-maxage response directive (see Section 5.2.2.9)
	and the cache is shared, or
	- contains a Cache Control Extension (see Section 5.2.3) that
	allows it to be cached, or
	- has a status code that is defined as cacheable by default (see
	Section 4.2.2), or
	- contains a public response directive (see Section 5.2.2.5).

**大概解释一下：**

一个响应满足以下条件，即可被缓存：
* 请求的方法必须能被缓存理解，而且必须是可以被缓存的方法。例如GET、HEAD方法可以被缓存，POST和PATCH如果设置了合适的Header(Content-Location)也可以被缓存，但PUT和DELETE方法不可被缓存。
* 响应码可以被缓存理解，以下这些响应码是可以被缓存的：200、203、204、206、300、301、404、405、410、414、501。
* 请求头和响应头中都没有指定 "no-store" 头部
* 对于共享缓存来说（代理服务器），在响应头中没有指定为 "private"
* 对于共享缓存来说（代理服务器），请求中没有 "Authorization" 头部
* 响应头中含有 "Expires"、"max-age"、"s-maxage"、"public"，或通过 "Cache Control Extension" 明确指明要缓存，或返回的响应码指明要缓存时

### 如何命中缓存

同样引用RFC7234原文做简要说明（原文：page8，第四章节）

When presented with a request, a cache MUST NOT reuse a stored
response, unless:
* The presented effective request URI (Section 5.5 of \[RFC7230\]) and
that of the stored response match, and
* the request method associated with the stored response allows it
to be used for the presented request, and
* selecting header fields nominated by the stored response (if any)
match those presented (see Section 4.1), and
* the presented request does not contain the no-cache pragma
(Section 5.4), nor the no-cache cache directive (Section 5.2.1),
unless the stored response is successfully validated
(Section 4.3), and
* the stored response does not contain the no-cache cache directive
(Section 5.2.2.2), unless it is successfully validated
(Section 4.3), and
* the stored response is either:
	- fresh (see Section 4.2), or
	- allowed to be served stale (see Section 4.2.4), or
	- successfully validated (see Section 4.3).

**大概解释一下：**

发起请求时，满足一下条件即可使用缓存：
* URI匹配，如果一个URI有多个缓存，则使用时间最近的
* 缓存的响应允许我们当前请求的METHOD使用缓存
* 头部匹配，指缓存的响应中，Vary指定的头部必须与本次请求的头部匹配
* 提交的请求不包含 "no-cache" 头部（Pragma和Cache-Control都不可包含）
* 缓存的响应不包含 "no-cache" 头部（Pragma和Cache-Control都不可包含）
* 缓存未过期，或允许使用过期缓存(max-stale)，或针对过期缓存已经到源服务器成功验证(源服务器响应304)

### Varying Response

这里对命中缓存时提到的头部匹配做一个介绍。`Vary` 是Http响应头的一个Header，他决定了一个请求在命中缓存时的Header匹配规则。当缓存的响应中指定了Vary时，新的请求必须满足Vary指定的所有Header规则才可使用缓存。例如：

![](http://image.feathers.top/image/HttpCache8.png)

上图中的这个js资源Response中的Vary指明，想要使用我这个缓存，必须验证 `Accept-Encoding` 头的值，再看Request Header中，请求的 `Accept-Encoding` 头的值为 `gzip, deflate`。所以，新的请求想要命中这个缓存，也必须带这个头才行。

再来看一个来自 Mozilla 官网的例子。

![](http://image.feathers.top/image/HttpCache9.png)

Client1向代理服务器Cache发起/doc的请求，Cache发现没有缓存，向源服务器Server请求，Server将结果响应给Cache，并带有一个Vary头部，指定要校验 `Content-Encoding` 头部，值为gzip的请求才能使用这份共享缓存。然后再响应给Client1。

Client2向代理服务器Cache发起/doc请求，并带有 `Accept-Encoding` 头，值为br。Cache服务器跟本地缓存校对，发现Vary校验不通过，因此不能使用共享缓存，再次向源服务器发起请求，此时源服务器会返回响应，并将请求的br加到限制条件中。再返回给Client2。

Client3向代理服务器Cache发起/doc请求，并带有 `Accept-Encoding` 头，值为br。Cache服务器跟本地缓存校对，发现Vary中指定的值包含br，校验通过，直接返回共享缓存，不再请求源服务器。

## 参考

* [极客时间 - Web协议详解与抓包实战](https://time.geekbang.org/course/intro/100026801?utm_term=pc_interstitial_259)
* [RFC7234](https://www.rfc-editor.org/rfc/pdfrfc/rfc7234.txt.pdf)
* [Cacheable](https://developer.mozilla.org/en-US/docs/Glossary/cacheable)
* [Http Caching](https://developer.mozilla.org/en-US/docs/Web/HTTP/Caching)
