

### 实验

#### 启动JVM，不添加JWarmup相关参数

JVM参数：

```text
-Xms4096M -Xmx4096M -XX:+UseG1GC -XX:+PrintGCApplicationStoppedTime
-XX:+PrintGCApplicationConcurrentTime -XX:+PrintSafepointStatistics
-XX:PrintSafepointStatisticsCount=1 -verbose:gc
-Xloggc:/data/coohua/logs/gc.log -XX:+UseGCLogFileRotation
-XX:GCLogFileSize=20M -XX:NumberOfGCLogFiles=12 -XX:+PrintGCDetails
-XX:+PrintGCDateStamps -XX:+PrintTenuringDistribution -XX:+PrintHeapAtGC
-Dahas.namespace=prod -Dproject.name=ad-web -XX:+UnlockExperimentalVMOptions
-XX:+UseCGroupMemoryLimitForHeap -XX:ParallelGCThreads=4
-XX:-CICompilerCountPerCPU -XX:CICompilerCount=2 -XX:ConcGCThreads=4
-XX:G1ConcRefinementThreads=4 -XX:+PreserveFramePointer
```

CPU配置：

```text
4 core
```

* 第一次

![dragonwell1](http://image.feathers.top/image/dragonwell1.png)

![dragonwell-接口性能](http://image.feathers.top/image/dragonwell-接口性能.png)

* 第二次

![dragonwell2](http://image.feathers.top/image/dragonwell2.png)

![dragonwell2-接口性能](http://image.feathers.top/image/dragonwell2-接口性能.png)


#### 使用CMS GC收集jwarmup profiling参数

JVM参数：

```text
-Xms4096M -Xmx4096M -XX:+UseConcMarkSweepGC -XX:+PrintGCApplicationStoppedTime
-XX:+PrintGCApplicationConcurrentTime -XX:+PrintSafepointStatistics
-XX:PrintSafepointStatisticsCount=1 -verbose:gc
-Xloggc:/data/coohua/logs/gc.log -XX:+UseGCLogFileRotation
-XX:GCLogFileSize=20M -XX:NumberOfGCLogFiles=12 -XX:+PrintGCDetails
-XX:+PrintGCDateStamps -XX:+PrintTenuringDistribution -XX:+PrintHeapAtGC
-Dahas.namespace=prod -Dproject.name=ad-web -XX:+UnlockExperimentalVMOptions
-XX:+UseCGroupMemoryLimitForHeap -XX:ParallelGCThreads=4
-XX:-CICompilerCountPerCPU -XX:CICompilerCount=2 -XX:ConcGCThreads=4
-XX:G1ConcRefinementThreads=4 -XX:+PreserveFramePointer
-XX:-ClassUnloading -XX:-CMSClassUnloadingEnabled -XX:-ClassUnloadingWithConcurrentMark
-XX:CompilationWarmUpLogfile=/data/coohua/logs/ad-web-impl-canary-0/jwarmup.log 
-XX:+CompilationWarmUpRecording -XX:CompilationWarmUpRecordTime=300
```

* 第一次

![dragonwell4](http://image.feathers.top/image/dragonwell4.png)

![dragonwell4-接口性能](http://image.feathers.top/image/dragonwell4-接口性能.png)


#### 启动JVM，使用dump的profiling信息，预热

JVM参数：

```text
-Xms4096M -Xmx4096M -XX:+UseConcMarkSweepGC -XX:+PrintGCApplicationStoppedTime
-XX:+PrintGCApplicationConcurrentTime -XX:+PrintSafepointStatistics
-XX:PrintSafepointStatisticsCount=1 -verbose:gc
-Xloggc:/data/coohua/logs/gc.log -XX:+UseGCLogFileRotation
-XX:GCLogFileSize=20M -XX:NumberOfGCLogFiles=12 -XX:+PrintGCDetails
-XX:+PrintGCDateStamps -XX:+PrintTenuringDistribution -XX:+PrintHeapAtGC
-Dahas.namespace=prod -Dproject.name=ad-web -XX:+UnlockExperimentalVMOptions
-XX:+UseCGroupMemoryLimitForHeap -XX:ParallelGCThreads=4
-XX:-CICompilerCountPerCPU -XX:CICompilerCount=2 -XX:ConcGCThreads=4
-XX:G1ConcRefinementThreads=4 -XX:+PreserveFramePointer
-XX:+CompilationWarmUp -XX:-TieredCompilation 
-XX:CompilationWarmUpLogfile=/data/coohua/logs/ad-web-impl-canary-0/jwarmup.log -XX:CompilationWarmUpDeoptTime=0
```

![dragonwell5](http://image.feathers.top/image/dragonwell5.png)

![dragonwell5-接口性能](http://image.feathers.top/image/dragonwell5-接口性能.png)

* error log

```text
2019-08-01 17:02:49.623 [http-nio-8080-exec-478] [ERROR] [1564650168212-123.196.14.87] [c.c.a.w.c.WebController] - Server error 500. org.apache.catalina.connector.ClientAbortException: java.io.IOException: Brok
en pipe
        at org.apache.catalina.connector.OutputBuffer.doFlush(OutputBuffer.java:298) ~[tomcat-embed-core-9.0.13.jar!/:9.0.13]
        at org.apache.catalina.connector.OutputBuffer.flush(OutputBuffer.java:261) ~[tomcat-embed-core-9.0.13.jar!/:9.0.13]
        at org.apache.catalina.connector.CoyoteOutputStream.flush(CoyoteOutputStream.java:118) ~[tomcat-embed-core-9.0.13.jar!/:9.0.13]
        at com.alibaba.fastjson.support.spring.FastJsonHttpMessageConverter.write(FastJsonHttpMessageConverter.java:218) ~[fastjson-1.2.23.jar!/:?]
        at org.springframework.web.servlet.mvc.method.annotation.AbstractMessageConverterMethodProcessor.writeWithMessageConverters(AbstractMessageConverterMethodProcessor.java:289) ~[spring-webmvc-5.1.3.RELEAS
E.jar!/:5.1.3.RELEASE]
        at org.springframework.web.servlet.mvc.method.annotation.RequestResponseBodyMethodProcessor.handleReturnValue(RequestResponseBodyMethodProcessor.java:180) ~[spring-webmvc-5.1.3.RELEASE.jar!/:5.1.3.RELEA
SE]
        at org.springframework.web.method.support.HandlerMethodReturnValueHandlerComposite.handleReturnValue(HandlerMethodReturnValueHandlerComposite.java:82) ~[spring-web-5.1.3.RELEASE.jar!/:5.1.3.RELEASE]
        at org.springframework.web.servlet.mvc.method.annotation.ServletInvocableHandlerMethod.invokeAndHandle(ServletInvocableHandlerMethod.java:119) ~[spring-webmvc-5.1.3.RELEASE.jar!/:5.1.3.RELEASE]
        at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.invokeHandlerMethod(RequestMappingHandlerAdapter.java:895) ~[spring-webmvc-5.1.3.RELEASE.jar!/:5.1.3.RELEASE]
        at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.handleInternal(RequestMappingHandlerAdapter.java:800) ~[spring-webmvc-5.1.3.RELEASE.jar!/:5.1.3.RELEASE]
        at org.springframework.web.servlet.mvc.method.AbstractHandlerMethodAdapter.handle(AbstractHandlerMethodAdapter.java:87) ~[spring-webmvc-5.1.3.RELEASE.jar!/:5.1.3.RELEASE]
        at org.springframework.web.servlet.DispatcherServlet.doDispatch(DispatcherServlet.java:1038) [spring-webmvc-5.1.3.RELEASE.jar!/:5.1.3.RELEASE]
        at org.springframework.web.servlet.DispatcherServlet.doService(DispatcherServlet.java:942) [spring-webmvc-5.1.3.RELEASE.jar!/:5.1.3.RELEASE]
        at org.springframework.web.servlet.FrameworkServlet.processRequest(FrameworkServlet.java:1005) [spring-webmvc-5.1.3.RELEASE.jar!/:5.1.3.RELEASE]
        at org.springframework.web.servlet.FrameworkServlet.doPost(FrameworkServlet.java:908) [spring-webmvc-5.1.3.RELEASE.jar!/:5.1.3.RELEASE]
        at javax.servlet.http.HttpServlet.service(HttpServlet.java:660) [tomcat-embed-core-9.0.13.jar!/:9.0.13]
        at org.springframework.web.servlet.FrameworkServlet.service(FrameworkServlet.java:882) [spring-webmvc-5.1.3.RELEASE.jar!/:5.1.3.RELEASE]
        at javax.servlet.http.HttpServlet.service(HttpServlet.java:741) [tomcat-embed-core-9.0.13.jar!/:9.0.13]
        at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:231) [tomcat-embed-core-9.0.13.jar!/:9.0.13]
        at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166) [tomcat-embed-core-9.0.13.jar!/:9.0.13]
        at org.apache.tomcat.websocket.server.WsFilter.doFilter(WsFilter.java:53) [tomcat-embed-websocket-9.0.13.jar!/:9.0.13]
        at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193) [tomcat-embed-core-9.0.13.jar!/:9.0.13]
        at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166) [tomcat-embed-core-9.0.13.jar!/:9.0.13]
        at org.springframework.boot.actuate.web.trace.servlet.HttpTraceFilter.doFilterInternal(HttpTraceFilter.java:90) [spring-boot-actuator-2.1.1.RELEASE.jar!/:2.1.1.RELEASE]
        at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107) [spring-web-5.1.3.RELEASE.jar!/:5.1.3.RELEASE]
        at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193) [tomcat-embed-core-9.0.13.jar!/:9.0.13]
        at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166) [tomcat-embed-core-9.0.13.jar!/:9.0.13]
        at com.coohua.caf.core.sentinel.SentinelHttpFilter.doFilter(SentinelHttpFilter.java:19) [caf-core-1.0.34.jar!/:1.0.34]
        at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193) [tomcat-embed-core-9.0.13.jar!/:9.0.13]
        at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166) [tomcat-embed-core-9.0.13.jar!/:9.0.13]
        at com.coohua.caf.core.metrics.ProfilerHttpFilter.doFilter(ProfilerHttpFilter.java:56) [caf-core-1.0.34.jar!/:1.0.34]
        at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193) [tomcat-embed-core-9.0.13.jar!/:9.0.13]
        at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166) [tomcat-embed-core-9.0.13.jar!/:9.0.13]
        at org.springframework.web.filter.RequestContextFilter.doFilterInternal(RequestContextFilter.java:99) [spring-web-5.1.3.RELEASE.jar!/:5.1.3.RELEASE]
        at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107) [spring-web-5.1.3.RELEASE.jar!/:5.1.3.RELEASE]
        at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193) [tomcat-embed-core-9.0.13.jar!/:9.0.13]
        at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166) [tomcat-embed-core-9.0.13.jar!/:9.0.13]
        at org.springframework.web.filter.FormContentFilter.doFilterInternal(FormContentFilter.java:92) [spring-web-5.1.3.RELEASE.jar!/:5.1.3.RELEASE]
        at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107) [spring-web-5.1.3.RELEASE.jar!/:5.1.3.RELEASE]
        at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193) [tomcat-embed-core-9.0.13.jar!/:9.0.13]
        at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166) [tomcat-embed-core-9.0.13.jar!/:9.0.13]
        at org.springframework.web.filter.HiddenHttpMethodFilter.doFilterInternal(HiddenHttpMethodFilter.java:93) [spring-web-5.1.3.RELEASE.jar!/:5.1.3.RELEASE]
        at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107) [spring-web-5.1.3.RELEASE.jar!/:5.1.3.RELEASE]
        at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193) [tomcat-embed-core-9.0.13.jar!/:9.0.13]
        at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166) [tomcat-embed-core-9.0.13.jar!/:9.0.13]
        at org.springframework.boot.actuate.metrics.web.servlet.WebMvcMetricsFilter.filterAndRecordMetrics(WebMvcMetricsFilter.java:117) [spring-boot-actuator-2.1.1.RELEASE.jar!/:2.1.1.RELEASE]
        at org.springframework.boot.actuate.metrics.web.servlet.WebMvcMetricsFilter.doFilterInternal(WebMvcMetricsFilter.java:106) [spring-boot-actuator-2.1.1.RELEASE.jar!/:2.1.1.RELEASE]
        at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107) [spring-web-5.1.3.RELEASE.jar!/:5.1.3.RELEASE]
        at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193) [tomcat-embed-core-9.0.13.jar!/:9.0.13]
        at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166) [tomcat-embed-core-9.0.13.jar!/:9.0.13]
        at org.springframework.web.filter.CharacterEncodingFilter.doFilterInternal(CharacterEncodingFilter.java:200) [spring-web-5.1.3.RELEASE.jar!/:5.1.3.RELEASE]
        at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107) [spring-web-5.1.3.RELEASE.jar!/:5.1.3.RELEASE]
        at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193) [tomcat-embed-core-9.0.13.jar!/:9.0.13]
        at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166) [tomcat-embed-core-9.0.13.jar!/:9.0.13]
        at org.apache.catalina.core.StandardWrapperValve.invoke(StandardWrapperValve.java:199) [tomcat-embed-core-9.0.13.jar!/:9.0.13]
        at org.apache.catalina.core.StandardContextValve.invoke(StandardContextValve.java:96) [tomcat-embed-core-9.0.13.jar!/:9.0.13]
        at org.apache.catalina.authenticator.AuthenticatorBase.invoke(AuthenticatorBase.java:490) [tomcat-embed-core-9.0.13.jar!/:9.0.13]
        at org.apache.catalina.core.StandardHostValve.invoke(StandardHostValve.java:139) [tomcat-embed-core-9.0.13.jar!/:9.0.13]
        at org.apache.catalina.valves.ErrorReportValve.invoke(ErrorReportValve.java:92) [tomcat-embed-core-9.0.13.jar!/:9.0.13]
        at org.apache.catalina.core.StandardEngineValve.invoke(StandardEngineValve.java:74) [tomcat-embed-core-9.0.13.jar!/:9.0.13]
        at org.apache.catalina.connector.CoyoteAdapter.service(CoyoteAdapter.java:343) [tomcat-embed-core-9.0.13.jar!/:9.0.13]
        at org.apache.coyote.http11.Http11Processor.service(Http11Processor.java:408) [tomcat-embed-core-9.0.13.jar!/:9.0.13]
        at org.apache.coyote.AbstractProcessorLight.process(AbstractProcessorLight.java:66) [tomcat-embed-core-9.0.13.jar!/:9.0.13]
        at org.apache.coyote.AbstractProtocol$ConnectionHandler.process(AbstractProtocol.java:791) [tomcat-embed-core-9.0.13.jar!/:9.0.13]
        at org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.doRun(NioEndpoint.java:1417) [tomcat-embed-core-9.0.13.jar!/:9.0.13]
        at org.apache.tomcat.util.net.SocketProcessorBase.run(SocketProcessorBase.java:49) [tomcat-embed-core-9.0.13.jar!/:9.0.13]
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149) [?:1.8.0_212]
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624) [?:1.8.0_212]
        at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61) [tomcat-embed-core-9.0.13.jar!/:9.0.13]
        at java.lang.Thread.run(Thread.java:748) [?:1.8.0_212]
Caused by: java.io.IOException: Broken pipe
        at sun.nio.ch.FileDispatcherImpl.write0(Native Method) ~[?:1.8.0_212]
        at sun.nio.ch.SocketDispatcher.write(SocketDispatcher.java:47) ~[?:1.8.0_212]
        at sun.nio.ch.IOUtil.writeFromNativeBuffer(IOUtil.java:93) ~[?:1.8.0_212]
        at sun.nio.ch.IOUtil.write(IOUtil.java:65) ~[?:1.8.0_212]
        at sun.nio.ch.SocketChannelImpl.write(SocketChannelImpl.java:471) ~[?:1.8.0_212]
        at org.apache.tomcat.util.net.NioChannel.write(NioChannel.java:134) ~[tomcat-embed-core-9.0.13.jar!/:9.0.13]
        at org.apache.tomcat.util.net.NioBlockingSelector.write(NioBlockingSelector.java:101) ~[tomcat-embed-core-9.0.13.jar!/:9.0.13]
        at org.apache.tomcat.util.net.NioSelectorPool.write(NioSelectorPool.java:157) ~[tomcat-embed-core-9.0.13.jar!/:9.0.13]
        at org.apache.tomcat.util.net.NioEndpoint$NioSocketWrapper.doWrite(NioEndpoint.java:1225) ~[tomcat-embed-core-9.0.13.jar!/:9.0.13]
        at org.apache.tomcat.util.net.SocketWrapperBase.doWrite(SocketWrapperBase.java:743) ~[tomcat-embed-core-9.0.13.jar!/:9.0.13]
        at org.apache.tomcat.util.net.SocketWrapperBase.flushBlocking(SocketWrapperBase.java:696) ~[tomcat-embed-core-9.0.13.jar!/:9.0.13]
        at org.apache.tomcat.util.net.SocketWrapperBase.flush(SocketWrapperBase.java:686) ~[tomcat-embed-core-9.0.13.jar!/:9.0.13]
        at org.apache.coyote.http11.Http11OutputBuffer$SocketOutputBuffer.flush(Http11OutputBuffer.java:553) ~[tomcat-embed-core-9.0.13.jar!/:9.0.13]
        at org.apache.coyote.http11.filters.IdentityOutputFilter.flush(IdentityOutputFilter.java:117) ~[tomcat-embed-core-9.0.13.jar!/:9.0.13]
        at org.apache.coyote.http11.Http11OutputBuffer.flush(Http11OutputBuffer.java:216) ~[tomcat-embed-core-9.0.13.jar!/:9.0.13]
        at org.apache.coyote.http11.Http11Processor.flush(Http11Processor.java:1149) ~[tomcat-embed-core-9.0.13.jar!/:9.0.13]
        at org.apache.coyote.AbstractProcessor.action(AbstractProcessor.java:394) ~[tomcat-embed-core-9.0.13.jar!/:9.0.13]
        at org.apache.coyote.Response.action(Response.java:209) ~[tomcat-embed-core-9.0.13.jar!/:9.0.13]
        at org.apache.catalina.connector.OutputBuffer.doFlush(OutputBuffer.java:294) ~[tomcat-embed-core-9.0.13.jar!/:9.0.13]
        ... 69 more
```



#### 启动JVM，添加JWarmup profiling收集参数

JVM参数：

```text
-Xms4096M -Xmx4096M -XX:+UseG1GC -XX:+PrintGCApplicationStoppedTime
-XX:+PrintGCApplicationConcurrentTime -XX:+PrintSafepointStatistics
-XX:PrintSafepointStatisticsCount=1 -verbose:gc
-Xloggc:/data/coohua/logs/gc.log -XX:+UseGCLogFileRotation
-XX:GCLogFileSize=20M -XX:NumberOfGCLogFiles=12 -XX:+PrintGCDetails
-XX:+PrintGCDateStamps -XX:+PrintTenuringDistribution -XX:+PrintHeapAtGC
-Dahas.namespace=prod -Dproject.name=ad-web -XX:+UnlockExperimentalVMOptions
-XX:+UseCGroupMemoryLimitForHeap -XX:ParallelGCThreads=4
-XX:-CICompilerCountPerCPU -XX:CICompilerCount=2 -XX:ConcGCThreads=4
-XX:G1ConcRefinementThreads=4 -XX:+PreserveFramePointer
-XX:-ClassUnloading -XX:-CMSClassUnloadingEnabled -XX:-ClassUnloadingWithConcurrentMark
-XX:CompilationWarmUpLogfile=/data/coohua/logs/ad-web-impl-canary-0/jwarmup.log 
-XX:+CompilationWarmUpRecording -XX:CompilationWarmUpRecordTime=300
```

CPU配置：

```text
4 core
```

* 第一次

![dragonwell3](http://image.feathers.top/image/dragonwell3.png)

![dragonwell3-接口性能](http://image.feathers.top/image/dragonwell3-接口性能.png)




### 参考

* [github](https://github.com/alibaba/dragonwell8)
* [独家 | 阿里 Dragonwell JDK 重磅发布 GA 版本：生产环境可用](https://www.infoq.cn/article/McfDp6lggF0fk_7xvsiQ)