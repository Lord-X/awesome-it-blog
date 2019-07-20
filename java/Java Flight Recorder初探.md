### 1 Java Flight Recorder是啥

#### 1.1 简介

Java Flight Recorder简称JFR，OpenJDK从11版本开始支持。它是一个低开销的数据收集框架，可用于在生产环境中分析Java应用和JVM运行状况及性能问题。

#### 1.2 JFR的背景

故障诊断、监控和profile收集分析是开发周期中不可缺少的一部分。但是很多问题都只会在高负载的生产环境中产生。此时就需要一个可以在生产环境中使用的监控工具，JFR由此而生。

JFR会从应用程序中记录运行时事件，同时也会记录JVM和OS的。记录的结果会存在一个单独的文件中，此文件可供开发工程师分析bug和性能问题。

同时JDK中也提供了可视化工具来分析这类文件。

#### 1.3 详述

JFR在JEP：167的Event-based JVM Tracing的基础上做了扩展。JEP167只将event简单的输出到stdout，而JFR提供了更高性能的基于二进制格式的event输出。

JFR在JDK中相关的模块如下：

```text
jdk.jfr 
    * API and internals
    * Requires only java.base (suitable for resource constrained devices)
    
jdk.management.jfr
    * JMX capabilities
    * Requires jdk.jfr and jdk.management
```

JFR有如下两种启动方式
* 增加JVM参数：-XX:StartFlightRecording
* 通过jcmd工具使用，用例如下：
    * jcmd <pid> JFR.start ：开始记录
    * jcmd <pid> JFR.dump filename=recording.jfr ：将记录文件dump下来
    * jcmd <pid> JFR.stop ：停止

dump下来的jfr文件可以通过jmc来分析。

#### 1.4 通过jcmd转储JFR文件

![JFR_dump](http://image.feathers.top/image/JFR的使用1.png)

就是下面这个文件了
![discovery.jfr](http://image.feathers.top/image/JFR的使用2.png)

### 2 通过jmc分析jfr

JDK11中已经移除了jmc工具包，从JDK11的what's new可以看出：

![jdk11_remove_jmc](http://image.feathers.top/image/jdk11_remove_jmc.png)

但是，跑在JDK11上的应用程序，dump出来的jfr用jmc6及以前的版本都是打不开的，需要最新的jcm7才能打开：

![JDK11JFR不适配jmc6及以前版本](http://image.feathers.top/image/JDK11JFR不适配jmc6及以前版本.png)

经过各种google，最终发现，目前还无法直接下载jmc7的二进制版，但可以自行build。build方式如下：

> https://github.com/JDKMissionControl/jmc

编译完成后，打开jmc，加载jfr文件，就可以看到下面的界面了。
![jmc加载jfr](http://image.feathers.top/image/20190720121145.png)




### 3 参考

* [JDK Documents - JEP 328: Flight Recorder](https://openjdk.java.net/jeps/328)
* [JDK11 - Introduction to JDK Flight Recorder（Video）](https://www.youtube.com/watch?v=_69wTZR6lis)
* [Java Mission Control - Now serving OpenJDK binaries too!](https://blogs.oracle.com/java-platform-group/java-mission-control-now-serving-openjdk-binaries-too)
* [JMC 7 Early-Access Builds](https://jdk.java.net/jmc/)
* [GitHub: JDKMissionControl/jmc](https://github.com/JDKMissionControl/jmc)
* [OpenJDK Wiki: jmc main](https://wiki.openjdk.java.net/display/jmc/Main)
* [jmc7 log](http://hg.openjdk.java.net/jmc/jmc7/)
* [Fetching and Building OpenJDK Mission Control](http://hirt.se/blog/?p=947)

