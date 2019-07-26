### 介绍

Ahead-of-Time Compilation，简称AOT编译，是在Java9中提供的一个功能，它能够事先将应用中或JDK中的字节码编译成机器码（提前做了即时编译器的事儿，但与C1、C2编译有很大差别），然后在启动应用时，使用这些编译好的机器码来加快应用启动速度，可以降低应用启动初期由即时编译器导致的CPU使用率飙高。

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
