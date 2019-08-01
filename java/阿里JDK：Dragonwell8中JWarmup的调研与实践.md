

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