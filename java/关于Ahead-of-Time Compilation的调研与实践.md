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

jaotc和启动JVM用的运行时参数也必须是一致的，在jaotc中通过-J来指定Java运行时需要的参数，例如：

```text
jaotc -J-XX:+UseParallelGC -J-XX:-UseCompressedOops --output libHelloWorld.so HelloWorld.class 
java -XX:+UseParallelGC -XX:-UseCompressedOops -XX:AOTLibrary=./libHelloWorld.so HelloWorld
```

这些启动参数会被记录在生成的AOT文件中，JVM加载AOT文件时会校验参数是否一致，如果不一致则不会加载这个AOT库，此时，如果-XX:+UseAOTStrictLoading参数开启了，JVM进程会直接退出。

最后提一下，jaotc并没有解决class依赖的问题，这些被依赖的类必须被添加到classpath中，否则在编译AOT的过程中会抛出一个ClassNotFoundException异常。


### 3 AOT的编译模式

AOT有两种编译模式，通过--compile-for-tiered标记控制。添加这个标记的话，意味着使用分层编译方式来生成AOTLibrary，否则使用普通方式编译。

* 在没有开启的情况下，代码会被静态编译成机器码，这个编译过程是没有profile收集的，并且他不会被JIT重新编译。
* 开启的情况下，会收集程序运行的profiling信息。其效果等同于C1编译器在Tier2层的profiling编译效果，如果代码在运行的过程中达到C1的Tier3层编译的阈值，会触发C1的重新编译，用来收集所有的profiling信息，这些信息将用于C2编译器的编译优化。

可以看出，AOT编译即使在开启分层编译模式的情况下，也只是能替代部分C1的编译工作，他无法顶替C2，最终还是要经过C2重新编译的。

所以，对于因为C2导致应用启动CPU飙高的应用来说，使用AOT的方式并不会提升应用启动的性能。


### 4 案例：生成并使用java.base模块的AOTLibrary

因为java.base中的method量过大（大概50000+个），所以在用jaotc生成时，要给足够的内存，通过下面方式生成：

```text
jaotc -J-XX:+UseCompressedOops -J-XX:+UseG1GC -J-Xmx4g --compile-for-tiered --info --compile-commands java.base-list.txt --output libjava.base-coop.so --module java.base
```

--compile-commands标记指定的文件可用于指定只编译哪些、或排除编译哪些方法。由于java.base模块中的一些方法会导致编译失败，所以通过java.base-list.txt文件将之排除，这个文件内容如下：

```text
cat java.base-list.txt

# jaotc: java.lang.StackOverflowError
exclude sun.util.resources.LocaleNames.getContents()[[Ljava/lang/Object;
exclude sun.util.resources.TimeZoneNames.getContents()[[Ljava/lang/Object;
exclude sun.util.resources.cldr.LocaleNames.getContents()[[Ljava/lang/Object;
exclude sun.util.resources..*.LocaleNames_.*.getContents\(\)\[\[Ljava/lang/Object;
exclude sun.util.resources..*.LocaleNames_.*_.*.getContents\(\)\[\[Ljava/lang/Object;
exclude sun.util.resources..*.TimeZoneNames_.*.getContents\(\)\[\[Ljava/lang/Object;
exclude sun.util.resources..*.TimeZoneNames_.*_.*.getContents\(\)\[\[Ljava/lang/Object;
# java.lang.Error: Trampoline must not be defined by the bootstrap classloader
exclude sun.reflect.misc.Trampoline.<clinit>()V
exclude sun.reflect.misc.Trampoline.invoke(Ljava/lang/reflect/Method;Ljava/lang/Object;[Ljava/lang/Object;)Ljava/lang/Object;
# JVM asserts
exclude com.sun.crypto.provider.AESWrapCipher.engineUnwrap([BLjava/lang/String;I)Ljava/security/Key;
exclude sun.security.ssl.*
exclude sun.net.RegisteredDomain.<clinit>()V
# Huge methods
exclude jdk.internal.module.SystemModules.descriptors()[Ljava/lang/module/ModuleDescriptor;
```

生成AOTLibrary之后，通过下面方式加载并使用AOTLibrary：

```text
java -XX:AOTLibrary=./libjava.base-coop.so,./libHelloWorld.so HelloWorld
```

> PS1：由于Java9之后会默认使用G1收集器，并且默认开启UseCompressedOops，所以启动时可以不用指定。
> 
> PS2：除了在启动时通过参数-XX:AOTLibrary指定加载的AOTLibrary之外，也可以将生成的.so文件放到$JAVA_HOME/lib下，JVM启动时会自动扫描加载。

### 5 Java中AOT相关的一些options

* -XX:+/-UseAOT

使用AOT编译的文件，默认开启。

* -XX:AOTLibrary=\<file\>

指定AOT文件，多个文件之间用英文半角逗号分隔(,)。

* -XX:+/-PrintAOT

在标准输出日志打印使用到的AOTLibrary中的class和method。

> 下面是一些具备诊断功能的flag，使用时需要优先开启-XX:+UnlockDiagnosticVMOptions

* -XX:+/-UseAOTStrictLoading

启动时，如果没有任何一个AOTLibrary与当前的JVM环境匹配上，则JVM进程直接退出。

这个可用于判定是否成功加载并使用了AOTLibrary编译的class或method。

> 此外，下面的一些参数可通过日志的形式打印AOT的使用情况，需要配合-Xlog参数

* aotclassfingerprint

当class指纹不匹配时，打印日志。

* aotclassload

当可用的class在AOTLibrary中找到时，打印日志。

* aotclassresolve

当解析AOT中的class成功或失败时，打印日志。

### 6 jaotc的usage

使用格式如下：

```text
jaotc <options> <name or list>
```

name表示的是class name或jar文件，list可以是class、modules、jar或包含class文件的目录。

另外，options可以有如下选项：

* --output \<file\>

输出的文件名，默认名称是"unnamed.so"。

* --class-name \<class names\>

待编译的java classes列表。

* --jar \<jar files\>

待编译的jar files列表。

* --module \<modules\>

待编译的java modules列表。

* --directory \<dirs\>

指定搜索目录，会从指定目录搜索待编译的文件。

* --search-path \<dirs\>

搜索指定目录下的文件。

* --compile-commands \<file\>

指定包含编译指令的文件，加载AOT时会执行这些指令进行文件过滤（exclude or compileOnly），用法如下：

```text
exclude sun.util.resources..*.TimeZoneNames_.*.getContents\(\)\[\[Ljava/lang/Object; 
exclude sun.security.ssl.* 
compileOnly java.lang.String.*
```

exclude用于排除指定的方法，被排除掉的不会被编译。

compileOnly用于指定只编译哪些方法。

* --compile-for-tiered

为分层编译生成带有profiling的代码，默认不生成。

* --compile-with-assertions

用Java断言生成代码，默认不开启。

* --compile-threads \<number\>

用于编译的线程数量，默认值：min(16, available_cpus)

* --ignore-errors

忽略所有class加载时抛出的异常，默认情况下，如果编译过程中抛出异常，会直接退出编译。

* --exit-on-error

编译发生error时，退出编译。默认情况下，会跳过编译失败的method，其他的method编译仍会继续。

* --info

打印编译时的信息。

* --verbose

打印更全的编译时信息，在--info开启的情况下有效。

* --debug

打印所有细节编译时的信息，在--info和--verbose同时开启的情况下有效。

* --help

打印jaotc的usage信息。

* --version

打印版本信息。

* -J\<flag\>

指定JVM运行时参数。


### 7 使用AOT的一些限制

* AOT最初在JDK9作为一个实验性的功能出现，并且只能运行在Linux x64系统、64位JVM环境、Parallel或G1环境下。
* AOT编译时和使用时必须使用相同的JVM参数。
* AOT的编译和使用必须在同一个系统环境下。
* 无法编译动态生成的java code，例如lambda表达式。
* AOT不支持使用用户自定义类加载器加载的class，因为在编译阶段无法知晓最终运行是会使用哪个类加载器加载这个class。


### 8 实验

---

```bash
# 使用AOT编译java.base模块
/Library/Java/JavaVirtualMachines/jdk-11.0.3.jdk/Contents/Home/bin/jaotc -J-XX:+UseCompressedOops -J-XX:+UseG1GC -J-Xmx4g --compile-for-tiered --info --compile-commands java.base-list.txt --output libjava.base-coop.so --module java.base
```

---

#### 8.1 第一次 
* JDK11不带任何参数正常启动
* 启动接口性能：

![JDK11启动性能](http://image.feathers.top/image/JDK11启动性能.png)

#### 8.2 第二次
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


#### 8.3 生成JFR，通过JMC进一步分析

通过下面的方式生成JFR信息，这条指令的意思是：在应用启动20秒后开始收集JFR，收集120秒。

```text
-XX:StartFlightRecording=delay=20s,duration=120s,name=myrecording,filename=./record.jfr,settings=profile
```

> PS: 关于JFR和JMC，可参考：[Java Flight Recorder初探](https://github.com/Lord-X/awesome-it-blog/blob/master/java/Java%20Flight%20Recorder%E5%88%9D%E6%8E%A2.md)

分下面三种情况对比，主要对比C2的编译情况：

* 没有AOT的JFR（收集1分钟）
* 没有分层编译AOT的JFR（收集2分钟）
* 分层编译AOT的JFR（收集2分钟）

##### 8.3.1 没有AOT的JFR

* C2线程CPU时间片的消耗

右侧黄色部分表示C2的CPU消耗。

![没有AOT的JFR-C2线程CPU时间片的消耗](http://image.feathers.top/image/没有AOT的JFR-C2线程CPU时间片的消耗.png)

* 方法编译耗时

![没有AOT的JFR-方法编译耗时](http://image.feathers.top/image/没有AOT的JFR-方法编译耗时.png)

##### 8.3.2 没有分层编译AOT的JFR

* C2线程CPU时间片的消耗

![没有分层编译AOT的JFR-C2线程CPU时间片消耗](http://image.feathers.top/image/没有分层编译AOT的JFR-C2线程CPU时间片消耗.png)

* 方法编译耗时

![没有分层编译AOT的JFR-方法编译耗时](http://image.feathers.top/image/没有分层编译AOT的JFR-方法编译耗时.png)

由于没有AOT的JFR收集了1分钟，所以这里的时间跨度是上面的两倍。等比缩小后，可以发现两次在CPU时间片消耗上差不多，但这一次收集时间更长，出现了多个编译时间较长的方法。

##### 8.3.3 分层编译AOT的JFR

* C2线程CPU时间片的消耗

![分层编译AOT的JFR-C2线程CPU时间片的消耗](http://image.feathers.top/image/分层编译AOT的JFR-C2线程CPU时间片的消耗.png)

* 方法编译耗时

![分层编译AOT的JFR-方法编译耗时](http://image.feathers.top/image/分层编译AOT的JFR-方法编译耗时.png)

可看出跟没有分层编译是，没有明显差别。这也验证了AOT并不能优化C2的编译时间。


#### 8.4 结论

由于我们应用启动时消耗CPU时间片最多的是C2编译器，对于AOT来说，正如上文描述的那样，他并不能降低C2的消耗，所以AOT无法解决我们这类场景的问题。

**思考：**

其实可以根据AOT编译的原理想一下他适用的场景。AOT是一个静态编译过程，他是离线的，非运行时编译，所以无法收集足够有效的profiling信息，所以他编译的结果肯定无法与C2相媲美。
同时官文中也说了，他可以大概等同于C1的Tier2编译的效果，在这一层上，会收集较少的profiling信息，然后AOT就直接将字节码编译成机器码。根据这个特点，他应该可以用于短时运行应用的启动性能优化。对于长期跑到服务器，他应该是无能为力的。


### 9 参考

* [JEP 295: Ahead-of-Time Compilation](https://openjdk.java.net/jeps/295)
* [解密新一代 Java JIT 编译器 Graal](https://www.infoq.cn/article/Graal-Java-JIT-Compiler)
