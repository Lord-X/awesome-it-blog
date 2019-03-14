近期线上有个接口响应延迟P99波动较大，后对其进行了优化。响应延迟折线图如下：

![优化前后对比](http://pnxjswhv3.bkt.clouddn.com/image/redis1.jpg)

在12月11号11点左右优化完成后，P99趋于平稳，平均在70ms左右。

下面来说一下优化过程。

### 1 思考接口的执行过程
这个接口一共会经过三个服务，最终返回给客户端。执行流程如下：

![服务结构](http://pnxjswhv3.bkt.clouddn.com/image/redis2.jpg)

按照箭头所示流程，先访问服务1，服务1的结果返回给接口层，在请求服务2，服务2请求服务3，然后将结果返回给接口层。


### 2 分析
然后分别观察了服务1、服务2、服务3，主要观察的指标如下：
* 服务对外响应延迟
* CPU负载
* 网络抖动

观察后，服务2和服务3的这几个指标都没啥问题。

服务2的对外响应延迟波动情况与接口的波动颇为相似，再针对服务2分析。服务2是个IO密集型的服务，平均QPS在3K左右。

主要的几个IO操作包括：
* 单点Redis的读取
* 集群Redis的读取
* 数据库的读取
* 两个http接口的拉取
* 一次其他服务的调用

集群Redis的响应很快，平均在5ms左右（加上来回的网络消耗），数据库在10ms左右，http接口只有偶尔的慢请求，其他服务的调用也没问题。

最后发现单点的Redis响应时间过长

![P99响应](http://pnxjswhv3.bkt.clouddn.com/image/redis3.jpg)

如图所示，服务2接受到的每次请求会访问三次这个单点redis，这三次加起来有接近100ms，然后针对这个单点redis进行分析。

发现这台redis的CPU有如下波动趋势

![CPU波动](http://pnxjswhv3.bkt.clouddn.com/image/redis4.jpg)

基本上每一分钟会波动一次。

马上反应过来是开启了bgsave引起的（基本1分钟bgsave一次），因为之前有过类似的经验，就直接关掉bgsave再观察

![关闭bgsave后的CPU波动](http://pnxjswhv3.bkt.clouddn.com/image/redis5.jpg)

至此，业务平稳下来。

### 3 解决方案
线上的bgsave不能一直关闭，万一出现故障，会造成大量数据丢失。

具体方案如下：
* 先开启这台机器的bgsave
* 申请一台从服务器，并从这台机器上同步数据
* 同步完成后，主节点关闭bgsave，从节点开启bgsave

这样一来，主节点的读写不再受bgsave影响，同时也能用从节点保证数据不丢失。


### 4 bgsave引起CPU波动原因探索
首先要说一下bgsave的执行机制。执行bgsave时（无论以哪种方式执行），会先fork出一个子进程来，由子进程把数据库的快照写入硬盘，父进程会继续处理客户端的请求。

所以在平时没有bgsave的时候，进程状态如下：

![无bgsave的进程状态](http://pnxjswhv3.bkt.clouddn.com/image/redis6.jpg)

bgsave时，进程状态如下：

![开启bgsave的进程状态](http://pnxjswhv3.bkt.clouddn.com/image/redis7.jpg)

最上面CPU占用100%的就是fork出来的子进程，在执行bgsave，同时他完全独占了一个CPU（上面的红框）。

所以得出结论，这个CPU的波动是正常的，每一个波峰都是子进程bgsave所致。

### 5 bgsave引起的接口相应延迟探索
关于fork，在redis官网有这么一段描述：

> RDB disadvantages
  * RDB is NOT good if you need to minimize the chance of data loss in case Redis stops working (for example after a power outage). You can configure different save points where an RDB is produced (for instance after at least five minutes and 100 writes against the data set, but you can have multiple save points). However you'll usually create an RDB snapshot every five minutes or more, so in case of Redis stopping working without a correct shutdown for any reason you should be prepared to lose the latest minutes of data.
  * RDB needs to fork() often in order to persist on disk using a child process. Fork() can be time consuming if the dataset is big, and may result in Redis to stop serving clients for some millisecond or even for one second if the dataset is very big and the CPU performance not great. AOF also needs to fork() but you can tune how often you want to rewrite your logs without any trade-off on durability.
  
这里说了RDB的劣势，第二点说明了fork会造成的问题。

大意是：RDB为了将数据持久化到硬盘，需要经常fork一个子进程出来。数据集如果过大的话，fork()的执行可能会非常耗时，如果数据集非常大的话，可能会导致Redis服务器产生几毫秒甚至几秒钟的拒绝服务，并且CPU的性能会急剧下降。

这个停顿的时间长短取决于redis所在的系统，对于真实硬件、VMWare虚拟机或者KVM虚拟机来说，Redis进程每占用1个GB的内存，fork子进程的时间就增加10-20ms，对于Xen虚拟机来说，Redis进程每占用1个GB的内存，fork子进程的时间需要增加200-300ms。

但对于一个访问量大的Redis来说，10-20ms已经是很长时间了（我们的redis占用了10个G左右内存，估计停顿时间在100ms左右）。

**至此，造成接口响应延迟的原因就明确了：**

由于redis是单进程运行的，在fork子进程时，如果耗时过多，造成服务器的停顿，导致redis无法继续处理请求，进一步就会导致向redis发请求的客户端全都hang住，接口响应变慢。

### 6 深入分析fork机制
知道原因后，来看一下redis执行bgsave的源码（fork部分）：

注释中分析了如果fork卡住，会造成的影响。

```java
// 执行bgsave
int rdbSaveBackground(char * filename, rdbSaveInfo * rsi) {
    pid_t childpid;
    long long start;
    if (server.aof_child_pid != -1 || server.rdb_child_pid != -1) return C_ERR;
    server.dirty_before_bgsave = server.dirty;
    server.lastbgsave_try = time(NULL);
    openChildInfoPipe();

    // 记录执行fork的起始时间，用于计算fork的耗时
    start = ustime();
    // 在这里执行fork ！！
    // 由此可见，如果fork卡住，下面执行父进程的else条件就会卡住，子进程的执行也需要fork完成后才会开始
    if ((childpid = fork()) == 0) {
        // fork()返回了等于0的值，说明执行成功，
        int retval;
        // 下面是子进程的执行过程
        /* Child */
        closeListeningSockets(0);
        redisSetProcTitle("redis-rdb-bgsave");
        // 子进程执行硬盘的写操作
        retval = rdbSave(filename, rsi);
        if (retval == C_OK) {
            size_t private_dirty = zmalloc_get_private_dirty( - 1);
            if (private_dirty) {
                serverLog(LL_NOTICE, "RDB: %zu MB of memory used by copy-on-write", private_dirty / (1024 * 1024));
            }
            server.child_info_data.cow_size = private_dirty;
            sendChildInfo(CHILD_INFO_TYPE_RDB);
        }
        // 子进程执行完毕退出，返回执行结果给父进程，0 - 成功，1 - 失败
        exitFromChild((retval == C_OK) ? 0 : 1);
    } else {
        // 下面是父进程的执行过程
        /* Parent */
        // 计算fork的执行时间
        server.stat_fork_time = ustime() - start;
        server.stat_fork_rate = (double) zmalloc_used_memory() * 1000000 / server.stat_fork_time / (1024 * 1024 * 1024);
        /* GB per second. */
        latencyAddSampleIfNeeded("fork", server.stat_fork_time / 1000);
        if (childpid == -1) { // fork出错，打印错误日志
            closeChildInfoPipe();
            server.lastbgsave_status = C_ERR;
            serverLog(LL_WARNING, "Can't save in background: fork: %s", strerror(errno));
            return C_ERR;
        }
        serverLog(LL_NOTICE, "Background saving started by pid %d", childpid);
        server.rdb_save_time_start = time(NULL);
        server.rdb_child_pid = childpid;
        server.rdb_child_type = RDB_CHILD_TYPE_DISK;
        updateDictResizePolicy();
        return C_OK;
    }
    return C_OK;
    /* unreached */
}
```

fork()方法返回值的描述：

> Return Value
>
> On success, the PID of the child process is returned in the parent, and 0 is returned in the child. On failure, -1 is returned in the parent, no child process is created, and errno is set appropriately.

意思是，如果fork成功，此进程的PID会返回给父进程，并且会给fork出的子进程返回一个0。如果fork失败，给父进程返回-1，没有子进程创建，并设置一个系统错误码。

由此可见，fork的执行流程如下：

![fork的执行流程](http://pnxjswhv3.bkt.clouddn.com/image/redis8.jpg)

再来看看Linux中关于fork()的注意事项。

> Notes
>
> Under Linux, fork() is implemented using copy-on-write pages, so the only penalty that it incurs is the time and memory required to duplicate the parent's page tables, and to create a unique task structure for the child.
> Since version 2.3.3, rather than invoking the kernel's fork() system call, the glibc fork() wrapper that is provided as part of the NPTL threading implementation invokes clone(2) with flags that provide the same effect as the traditional system call. (A call to fork() is equivalent to a call to clone(2) specifying flags as just SIGCHLD.) The glibc wrapper invokes any fork handlers that have been established using pthread_atfork(3).

第一段描述了fork()的一些问题。大意如下：

在Linux系统下，fork()通过copy-on-write策略实现，因此，他会带来的问题是：复制父进程和为子进程创建唯一的进程结构所需要的时间和内存。

### 7 补充-进程的内存模型
系统内核会为每一个进程开辟一块虚拟内存空间，其分布如下

![虚拟内存空间图示](http://pnxjswhv3.bkt.clouddn.com/image/redis9.jpg)

fork的子进程相当于父进程的一个clone，可见，如果父进程中数据量比较多的话，clone的耗时会比较长。

### 参考文档
* Redis in Action. Josiah L. Carison
* Redis官网：Redis Persistence, 链接：https://redis.io/topics/persistence
* Redis源码：rdb.c, 链接：https://github.com/antirez/redis/blob/unstable/src/rdb.c
* Linux man page: fork(), 链接：https://linux.die.net/man/2/fork