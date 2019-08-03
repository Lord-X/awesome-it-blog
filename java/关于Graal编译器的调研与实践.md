### 0 介绍

Graal是一个款基于Java的JIT编译器，是JDK9中实验性功能：AOT编译的基础。自Java10以后，他作为一个实验性的JIT编译器被添加到HotSpot VM中。
目前需要在Linux/x64环境下使用。Graal基于JDK9中的JVM编译接口（JVMCI），由于Graal已经继承在JDK中，可以通过以下方式开启他：

```text
# 开启Graal编译器，替换C2：
-XX:+UnlockExperimentalVMOptions -XX:+UseJVMCICompiler
```

在传统情况下，JVM与JIT编译器是紧耦合的，更新JIT编译器需要重新编译整个JVM。这对于发展较快的Graal是一个很大的限制。

为了解决这个问题，在JDK9中提供了JVMCI，JVMCI会把下面三个功能抽象成Java层面的接口[8]：
* 响应编译请求。
* 获取编译所需的类、方法等元数据信息以及程序执行状态的profiling信息。
* 将生成的机器码部署到CodeCache里。

这样只要JVMCI不变，只需替换Graal的jar包即可完成Graal的升级。

### 1 为什么用Java写一个即时编译器



### Graal对应用启动性能的影响

应用启动时，Graal本身需要被即时编译，会抢占资源，导致应用启动性能下降。



### 参考

* [1] [Understanding How Graal Works - a Java JIT Compiler Written in Java](https://chrisseaton.com/truffleruby/jokerconf17/)
* [2] [Github - Graal](https://github.com/oracle/graal)
* [3] [OpenJDK - Graal Project](http://openjdk.java.net/projects/graal/)
* [4] [JEP 317: Experimental Java-Based JIT Compiler](https://openjdk.java.net/jeps/317)
* [5] [GraalVM Official](https://www.graalvm.org/)
* [6] [maxine-vm](https://community.oracle.com/community/groundbreakers/java/java_hotspot_virtual_machine/maxine-vm)
* [7] [深入浅出 Java 10 的实验性 JIT 编译器 Graal](https://www.infoq.cn/article/java-10-jit-compiler-graal)
* [8] [极客时间-深入拆解Java虚拟机-Graal:用Java编译Java by 郑雨迪](http://gk.link/a/105dI)
* [9] [JEP 243: Java-Level JVM Compiler Interface](http://openjdk.java.net/jeps/243)