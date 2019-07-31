### 0 介绍

Graal是一个款基于Java的JIT编译器，是JDK9中实验性功能：AOT编译的基础。自Java10以后，他作为一个实验性的JIT编译器被添加到HotSpot VM中。
目前需要在Linux/x64环境下使用。Graal基于JDK9中的JVM编译接口（JVMCI），由于Graal已经继承在JDK中，可以通过以下方式开启他：

```text
# 开启Graal编译器，替换C2：
-XX:+UnlockExperimentalVMOptions -XX:+UseJVMCICompiler
```




### 参考

* [Understanding How Graal Works - a Java JIT Compiler Written in Java](https://chrisseaton.com/truffleruby/jokerconf17/)
* [Github - Graal](https://github.com/oracle/graal)
* [OpenJDK - Graal Project](http://openjdk.java.net/projects/graal/)
* [JEP 317: Experimental Java-Based JIT Compiler](https://openjdk.java.net/jeps/317)
* [GraalVM Official](https://www.graalvm.org/)
* [maxine-vm](https://community.oracle.com/community/groundbreakers/java/java_hotspot_virtual_machine/maxine-vm)
* [深入浅出 Java 10 的实验性 JIT 编译器 Graal](https://www.infoq.cn/article/java-10-jit-compiler-graal)
* [极客时间-深入拆解Java虚拟机-Graal:用Java编译Java by 郑雨迪](http://gk.link/a/105dI)