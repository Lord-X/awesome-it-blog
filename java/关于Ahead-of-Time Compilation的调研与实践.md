### 0 介绍

Ahead-of-Time Compilation，简称AOT编译，是在Java9中提供的一个功能，它能够事先将应用中或JDK中的字节码编译成机器码（提前做了即时编译器的事儿，但与C1、C2编译有很大差别），然后在启动应用时，使用这些编译好的机器码来加快应用启动速度，可以降低应用启动初期由即时编译器导致的CPU使用率飙高。


### 1 AOT Quick Start

#### 1.1 生成AOT Library
AOT通过jaotc工具编译（在$JAVA_HOME/bin下），可以.class、java module、jar等为单位进行编译，编译结果为一个.so文件。jaotc是通过Graal编译器生成机器码的。

例如，通过下面方式将一个class文件编译成AOTLibrary：

```text
jaotc --output libHelloWorld.so HelloWorld.class
```

通过下面的方式来编译java.base模块：

```text
jaotc --output libjava.base.so --module java.base
```

#### 1.2 使用AOT Library

生成的.so文件使用起来非常方便，可通过以下两种方式使用：
* 将.so文件放到$JAVA_HOME/lib下，JVM启动时会自动加载
* 添加参数-XX:AOTLibrary，指定使用哪个AOTLibrary

第二种方式，可通过如下方式使用：

```text
java -XX:AOTLibrary=./libHelloWorld.so,./libjava.base.so HelloWorld
```

### 2 JVM对AOTLibrary的管理

JVM将编译好的AOTLibrary当做CodeCache的一个扩展。当一个Java class被加载时，JVM会先在AOTLibrary中看看有没有编译好的、与之匹配的method，如果有就直接使用了。

但我们的代码可能是经常变化的，JVM就需要识别这些变化的代码，并不再从AOTLibrary中加载这些类。这个功能是通过class指纹实现的。在jaotc编译AOTLibrary时，会同时生成每个class的指纹，并存储在编译而成的AOTLibrary中。

然后每次尝试从AOTLibrary中加载时，会比较class的指纹，从而实现这个功能。

此外，生成AOTLibrary时，jaotc编译使用的JDK版本和用于Java应用启动的JVM版本必须是一致的。jaotc后，JDK版本会记录在AOTLibrary中，在AOT被JVM加载时会做检查，如果版本不同，会拒绝加载。

最后提一下，jaotc并没有解决class依赖的问题，这些被依赖的类必须被添加到classpath中，否则在编译AOT的过程中会抛出一个ClassNotFoundException异常。

---

```bash
# 使用AOT编译java.base模块
/Library/Java/JavaVirtualMachines/jdk-11.0.3.jdk/Contents/Home/bin/jaotc -J-XX:+UseCompressedOops -J-XX:+UseG1GC -J-Xmx4g --compile-for-tiered --info --compile-commands java.base-list.txt --output libjava.base-coop.so --module java.base
```

---

## 第一次 
* JDK11不带任何参数正常启动
* 启动接口性能：

![JDK11启动性能](http://image.feathers.top/image/JDK11启动性能.png)

## 第二次
* 使用jaotc编译AOTLibrary（带分层编译），只编译java.base：

```text
jaotc --output libjava.base.tiered.so --compile-for-tiered --module java.base
```

* 使用AOT启动

```text
-XX:AOTLibrary=/data/coohua/logs/libjava.base.tiered.so
```

* 启动性能
![分层编译AOT启动](http://image.feathers.top/image/分层编译AOT启动.png)

* CPU
![分层编译AOT启动CPU消耗](http://image.feathers.top/image/分层编译AOT启动CPU消耗.png)
主要在C2编译器上。

* 与没有AOT的性能对比

![分层编译AOT接口性能对比](http://image.feathers.top/image/分层编译AOT接口性能对比.png)

看上去好了一些。




```text
-XX:StartFlightRecording=delay=20s,duration=60s,name=myrecording,filename=./record.jfr,settings=profile
```


没有AOT的JFR

分层编译AOT的JFR

没有分层编译AOT的JFR




### 参考

* [JEP 295: Ahead-of-Time Compilation](https://openjdk.java.net/jeps/295)
* [解密新一代 Java JIT 编译器 Graal](https://www.infoq.cn/article/Graal-Java-JIT-Compiler)
