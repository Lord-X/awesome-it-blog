### 介绍

Ahead-of-Time Compilation，简称AOT编译，是在Java9中提供的一个功能，它能够事先将应用中或JDK中的字节码编译成机器码（提前做了即时编译器的事儿，但与C1、C2编译有很大差别），然后在启动应用时，使用这些编译好的机器码来加快应用启动速度，可以降低应用启动初期由即时编译器导致的CPU使用率飙高。



```bash
# 使用AOT编译java.base模块
/Library/Java/JavaVirtualMachines/jdk-11.0.3.jdk/Contents/Home/bin/jaotc -J-XX:+UseCompressedOops -J-XX:+UseG1GC -J-Xmx4g --compile-for-tiered --info --compile-commands java.base-list.txt --output libjava.base-coop.so --module java.base
```


### 参考

* [JEP 295: Ahead-of-Time Compilation](https://openjdk.java.net/jeps/295)
* [解密新一代 Java JIT 编译器 Graal](https://www.infoq.cn/article/Graal-Java-JIT-Compiler)
