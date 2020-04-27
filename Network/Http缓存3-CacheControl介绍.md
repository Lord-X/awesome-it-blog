## Http缓存：Cache-Control介绍

Cache-Control头部既可以包含在请求头中，也可以包含在响应头中。其值的表示方式有两种，一种是描述一个特定的值，另一种是直接打一个标签标明某项功能。下面举例说明：

* 描述一个特定的值
Cache-Control: max-age=4152551

* 标签
Cache-Control: no-cache

请求头中，Cache-Control主要包含以下这些属性：

|属性|含义|
|----|----|
|max-age|告诉服务器，客户端不会接收age超出max-age的缓存，如果超出了，Server(如代理服务器)端需要重新从源服务器获取|
|max-stale|后面可以有值也可以没有，当有值的时候，表示即使Server端(如代理服务器)的缓存已过期，但过期秒数没有超出max-stale秒，客户端仍然要使用这个缓存；如果后面没有值，表明无论Server端的缓存过期多久，客户端都要使用他|
|min-fresh|当服务器端缓存的age超出min-fresh秒后，客户端才会使用，如果age小于min-fresh，要求Server端从源服务器获取资源|
|no-cache|表示不使用缓存，Server端也不能使用缓存|
|no-store|告诉各代理服务器不要将源服务器的资源缓存(实际有很多代理服务器不遵守这个约定)|
|no-transform|告诉代理服务器不要修改包体的内容|
|only-if-cached|告诉服务器只当有缓存时才返回，如果没缓存，则返回504|
|cache-extension|自定义扩展值，如果服务器不识别，将被忽略|

响应头中，Cache-Control主要包含以下这些属性：

|属性|含义|
|----|----|
|must-revalidate|告知客户端，如果缓存过期，必须向服务器验证后方可使用，以防有些客户端即使缓存过期了，也继续使用|
|proxy-revalidate|与must-revalidate类似，告知下游代理服务器，如果缓存过期，必须向服务器验证后方可使用|
|no-cache|告知客户端不能直接使用缓存，必须先到源服务器验证，如果no-cache后指定头部，则若客户端的后续请求及响应中不含有这些头则可以直接使用缓存|
|max-age|告诉客户端如果已缓存时间(age)超出了max-age指定的秒数，则缓存过期|
|s-maxage|与max-age类似，但只针对于共享缓存(通常是代理服务器中的缓存)，优先级高于max-age和Expires|
|public|告诉客户端，无论是私有缓存还是共享缓存，都可以将此响应缓存|
|private|表示该响应不能被代理服务器作为共享缓存使用|
|no-store|告诉所有下游节点不能对此响应缓存(包括代理服务器和客户端)|
|no-transform|告诉代理服务器不能修改响应包体的内容|

## 参考

* [极客时间 - Web协议详解与抓包实战](https://time.geekbang.org/course/intro/100026801?utm_term=pc_interstitial_259)
* [RFC7234](https://www.rfc-editor.org/rfc/pdfrfc/rfc7234.txt.pdf)
* [Cacheable](https://developer.mozilla.org/en-US/docs/Glossary/cacheable)
* [Http Caching](https://developer.mozilla.org/en-US/docs/Web/HTTP/Caching)