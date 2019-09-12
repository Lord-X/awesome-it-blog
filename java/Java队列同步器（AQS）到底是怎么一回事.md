## 开源推荐
推荐一款一站式性能监控工具（开源项目）

[Pepper-Metrics](https://github.com/zrbcool/pepper-metrics)是跟一位同事一起开发的开源组件，主要功能是通过比较轻量的方式与常用开源组件（jedis/mybatis/motan/dubbo/servlet）集成，收集并计算metrics，并支持输出到日志及转换成多种时序数据库兼容数据格式，配套的grafana dashboard友好的进行展示。项目当中原理文档齐全，且全部基于SPI设计的可扩展式架构，方便的开发新插件。另有一个基于docker-compose的独立demo项目可以快速启动一套demo示例查看效果[https://github.com/zrbcool/pepper-metrics-demo](https://github.com/zrbcool/pepper-metrics-demo)。如果大家觉得有用的话，麻烦给个star，也欢迎大家参与开发，谢谢：）

---

## 进入正题...

### 0 简介
队列同步器AbstractQueuedSynchronizer（后面简称AQS）是实现锁和有关同步器的一个基础框架。

在JDK5中，Doug Lea在并发包中加入了大量的同步工具，例如重入锁（ReentrantLock）、读写锁（ReentrantReadWriteLock）、信号量（Semaphore）、CountDownLatch等，都是基于AQS的。

其内部通过一个被标识为volatile的名为state的变量来控制多个线程之间的同步状态。多个线程之间可以通过AQS来独占式或共享式的抢占资源。

基于AQS，可以很方便的实现Java中不具备的功能。

例如，在锁这个问题上，Java中提供的是synchronized关键字，用这个关键字可以很方便的实现多个线程之间的同步。但这个关键字也有很多缺陷，比如：
* 他不支持超时的获取锁，一个线程一旦没有从synchronized上获取锁，就会卡在这里，没有机会逃脱。所以通常由synchronized造成的死锁是无解的。
* 不可响应中断。
* 不能尝试获取锁。如果尝试获取时没获取到，立刻返回，synchronized不具备这一特性。

而ReentrantLock基于AQS将上述几点都做到了。

### 1 核心结构
从AbstractQueuedSynchronizer的名字可以看出，AQS中一定是基于队列实现的（Queue）。在AQS内部，是通过链表实现的队列。链表的每个元素是其内部类Node的一个实现。然后AQS通过实例变量head指向队列的头，通过实例变量tail指向队列的尾。

其源码定义如下：

```java
/**
 * Head of the wait queue, lazily initialized.  Except for
 * initialization, it is modified only via method setHead.  Note:
 * If head exists, its waitStatus is guaranteed not to be
 * CANCELLED.
 */
private transient volatile Node head;

/**
 * Tail of the wait queue, lazily initialized.  Modified only via
 * method enq to add new wait node.
 */
private transient volatile Node tail;

/**
 * The synchronization state.
 */
private volatile int state;


static final class Node {

	/** 标识为共享式 */
    static final Node SHARED = new Node();
    /** 标识为独占式 */
    static final Node EXCLUSIVE = null;

	/** 同步队列中等待的线程等待超时或被中断，需要从等待队列中取消等待，进入该状态的节点状态将不再变化 */
    static final int CANCELLED =  1;

    /** 当前节点的后继节点处于等待状态，且当前节点释放了同步状态，需要通过unpark唤醒后继节点，让其继续运行 */
    static final int SIGNAL    = -1;

    /** 当前节点等待在某一Condition上，当其他线程调用这个Conditino的signal方法后，该节点将从等待队列恢复到同步队列中，使其有机会获取同步状态 */
    static final int CONDITION = -2;

    /** 表示下一次共享式同步状态获取状态将无条件的传播下去 */
    static final int PROPAGATE = -3;

	/* 当前节点的等待状态，取值为上述几个常量之一，另外，值为0表示初始状态 */
    volatile int waitStatus;

    /* 前驱节点 */
    volatile Node prev;

    /* 后继节点 */
    volatile Node next;

    /* 等待获取同步状态的线程 */
    volatile Thread thread;

    /* 等待队列中的后继节点 */
    Node nextWaiter;
    
    // ...
}
```

当线程通过AQS获取同步状态时，AQS会将当前线程封装到Node内部，并入队。所以在多个线程并发获取同步状态时，AQS内部会持有如下结构的队列：

![AQS内部队列](https://github.com/Lord-X/awesome-it-blog/blob/master/images/java/Java%E9%98%9F%E5%88%97%E5%90%8C%E6%AD%A5%E5%99%A8%EF%BC%88AQS%EF%BC%89%E5%88%B0%E5%BA%95%E6%98%AF%E6%80%8E%E4%B9%88%E4%B8%80%E5%9B%9E%E4%BA%8B/AQS%E5%86%85%E9%83%A8%E9%98%9F%E5%88%97.png?raw=true)

下文会基于这个队列模型，说明一下线程在AQS中获取同步状态时的原理。


### 2 实现原理

从AQS的名字可以看出来，作者是希望AQS作为一个基类来向外提供服务的（以Abstract标识）。所以通常AQS是以继承的方式使用的。

AQS提供了几个模板方法供实现类自己实现定制功能。

这几个方法是：
* boolean tryAcquire(int arg)：独占式的获取同步状态，通常通过以CAS的方式修改state的值来实现特定功能。
* boolean tryRelease(int arg)：独占式的释放同步状态，通常也是修改state的值。
* int tryAcquireShared(int arg)：共享式的获取同步状态，返回值>=0表示成功，否则失败。
* boolean tryReleaseShared(int arg)：共享式的释放同步状态，同样通过修改state值来实现。
* boolean isHeldExclusively()：表示AQS是否被当前线程独占。

这几个方法的默认实现都会抛出UnsupportedOperationException异常。

目前我们不用关心这几个方法，只要明白其内部是通过控制state的值来管理同步状态即可。

#### 2.1 同步状态的获取
通常，实现类会优先尝试修改state的值，来获取同步状态。例如，如果某个线程成功的将state的值从0修改为1，表示成功的获取了同步状态。
这个修改的过程是通过CAS完成的，所以可以保证线程安全。

反之，如果修改state失败，则会将当前线程加入到AQS的队列中，并阻塞线程。

AQS内部提供了三个方法来修改state的状态，其源码如下：

```java
/**
 * Returns the current value of synchronization state.
 * This operation has memory semantics of a {@code volatile} read.
 * @return current state value
 */
protected final int getState() {
    return state;
}

/**
 * Sets the value of synchronization state.
 * This operation has memory semantics of a {@code volatile} write.
 * @param newState the new state value
 */
protected final void setState(int newState) {
    state = newState;
}

/**
 * Atomically sets synchronization state to the given updated
 * value if the current state value equals the expected value.
 * This operation has memory semantics of a {@code volatile} read
 * and write.
 *
 * @param expect the expected value
 * @param update the new value
 * @return {@code true} if successful. False return indicates that the actual
 *         value was not equal to the expected value.
 */
protected final boolean compareAndSetState(int expect, int update) {
    // See below for intrinsics setup to support this
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
```

#### 2.2同步队列

如上文所述，AQS内部实际上是一个FIFO的双端队列，当线程获取同步状态失败时，就会构建一个Node并添加到队列尾部（此过程是线程安全的，CAS实现），并阻塞当前线程（通过LockSupport.park()方法）；
当释放同步状态时，AQS会先判断head节点是否为null，如果不是null，说明有等待同步状态的线程，就会尝试唤醒head节点，使其重新竞争同步状态。

#### 2.3 独占式同步状态的获取
独占式的意思就是说同一时间只能有一个线程获得同步状态。

AQS会先尝试调用实现类的tryAcquire方法获取同步状态，如果获取失败，会尝试将其封装为Node节点添加到同步队列尾部。

独占式同步状态的获取通过AQS的acquire方法实现。其源码如下：

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```
这个方法会先尝试获取一次同步状态（tryAcquire），如果获取失败，会通过addWaiter方法将当前线程加入到同步队列。
并在acquireQueued方法中将当前线程阻塞（LockSupport.park()），并进入自旋状态，以获取同步状态。

下面具体看一下他是如何构建Node并将其添加到队尾的。
首先是addWaiter：

```java
/**
 * Creates and enqueues node for current thread and given mode.
 *
 * @param mode Node.EXCLUSIVE for exclusive, Node.SHARED for shared
 * @return the new node
 */
private Node addWaiter(Node mode) {
    // mode = Node.EXCLUSIVE，表示是独占模式
    Node node = new Node(Thread.currentThread(), mode);
    // 先快速的通过CAS的方式将Node添加到队尾，如果失败，再进入enq方法通过无限循环添加
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}

/**
 * Inserts node into queue, initializing if necessary. See picture above.
 * @param node the node to insert
 * @return node's predecessor
 */
private Node enq(final Node node) {
    // 无限循环的将node添加到队尾，保证能添加成功
    /*
    注意：如果是首次向队列中添加Node，那么调addWaiter方法时，tail还是null，所以addWaiter方法不会设置成功，会直接进入这个方法
    进入这个方法后，由于tail仍然是null，所以会走到第一个if里面，这是会创建一个空的Node出来作为头结点
    然后再次循环，此时tail不是null了，会进入else的代码中，这时，才会将需要add的Node添加到队列尾部。
    也就是说，首次创建队列时，会默认加一个空的头结点。
     */
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```

再看下acquireQueued方法：

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        // 进入自旋，不断的获取同步状态
        for (;;) {
            // 获取node在队列中的前驱节点
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                // 如果成功进入到这块代码，说明成功的获取了同步状态
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            // 获取不成功，调用LockSupport.park()方法将当前线程阻塞
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

shouldParkAfterFailedAcquire方法用户判断是否需要阻塞当前线程，方法内会操作当前队尾节点的前驱节点的waitStatus，并依据waitStatus判断是否需要park。

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL) // Node.SIGNAL == -1
        /*
         * 表明当前节点需要其他线程的唤醒才能继续执行，此时可以安全的park。
         */
        return true;
    if (ws > 0) {
        /*
         * Predecessor was cancelled. Skip over predecessors and
         * indicate retry.
         */
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        /*
         * 如果一个节点是初始状态，即waitStatus=0时，
         * 将前驱节点的waitStatus设置为-1，表明其需要别的线程唤醒才能继续执行
         */
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```

当shouldParkAfterFailedAcquire方法判断当前节点需要被park时，会调用parkAndCheckInterrupt将其阻塞：

```java
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();
}
```

#### 2.4 独占式同步状态的释放
独占式的同步状态释放，在AQS中是通过release()方法实现的。此方法源码如下：

```java
public final boolean release(int arg) {
    // 尝试调用实现类的tryRelease方法来修改同步状态（state）
    if (tryRelease(arg)) {
        Node h = head;
        /*
        1.如果head节点是null，表示没有其他线程竞争同步状态，直接返回释放成功
        2.如果head节点不是null，表明有竞争。通过unparkSuccessor方法，通过unpark方法唤醒head节点的next节点。使其重新尝试竞争同步状态。
         */
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

unparkSuccessor方法会唤醒head节点的next节点，使其可以重新竞争同步状态：

```java
private void unparkSuccessor(Node node) {
    /*
     * 如果waitStatus的值是负数，例如：-1（等待signal）
     * 则将其值还原为0
     */
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    /*
     * 获取头结点的next节点，如果非空，则unpark他
     */
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);
}
```

#### 2.5 独占式同步状态获取与释放-图示
下面会通过画图方式展示一下源码中的过程，首先我们假设tryAcquire的实现如下：

```java
boolean tryAcquire(int acquires) {
    return compareAndSetState(0, acquires);
}
```
参数acquires固定传1，意为：通过CAS，如果成功将state的值从0修改为1，表示获取同步状态成功，否则失败，需要加入同步队列。

假设tryRelease的实现如下：

```java
boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (c == 0) {
        setState(c);
        return true;
    }
    return false;
}
```
参数releases固定传1，意为：如果当前state-1=0，视为释放成功，其他线程可竞争同步状态。

假设有三个线程并发获取同步状态，标识为t1、t2、t3，三个线程同时通过acquire方法修改state值。

假设t1修改成功，t2和t3修改失败。

t1修改成功之后，将state值变为1，并直接返回。此时head和tail都是空，所以同步队列也是空的。此时同步队列状态如下：

![t1成功修改state状态](https://github.com/Lord-X/awesome-it-blog/blob/master/images/java/Java%E9%98%9F%E5%88%97%E5%90%8C%E6%AD%A5%E5%99%A8%EF%BC%88AQS%EF%BC%89%E5%88%B0%E5%BA%95%E6%98%AF%E6%80%8E%E4%B9%88%E4%B8%80%E5%9B%9E%E4%BA%8B/AQS%E7%BA%BF%E7%A8%8Bt1%E8%8E%B7%E5%8F%96%E5%90%8C%E6%AD%A5%E7%8A%B6%E6%80%81%E6%88%90%E5%8A%9F.png?raw=true)

t2线程竞争同步状态失败，加入到同步队列中：

![t2加入同步队列](https://github.com/Lord-X/awesome-it-blog/blob/master/images/java/Java%E9%98%9F%E5%88%97%E5%90%8C%E6%AD%A5%E5%99%A8%EF%BC%88AQS%EF%BC%89%E5%88%B0%E5%BA%95%E6%98%AF%E6%80%8E%E4%B9%88%E4%B8%80%E5%9B%9E%E4%BA%8B/t2%E5%8A%A0%E5%85%A5%E5%90%8C%E6%AD%A5%E9%98%9F%E5%88%97.png?raw=true)

t3线程竞争同步状态失败，加入到同步队列中：

![t3加入同步队列](https://github.com/Lord-X/awesome-it-blog/blob/master/images/java/Java%E9%98%9F%E5%88%97%E5%90%8C%E6%AD%A5%E5%99%A8%EF%BC%88AQS%EF%BC%89%E5%88%B0%E5%BA%95%E6%98%AF%E6%80%8E%E4%B9%88%E4%B8%80%E5%9B%9E%E4%BA%8B/t3%E5%8A%A0%E5%85%A5%E5%90%8C%E6%AD%A5%E9%98%9F%E5%88%97.png?raw=true)

t1线程执行完毕，释放资源。
先将state还原为0，再unpark头结点的next节点（t2节点），使之重获同步状态的竞争资格。

![t1释放同步状态](https://github.com/Lord-X/awesome-it-blog/blob/master/images/java/Java%E9%98%9F%E5%88%97%E5%90%8C%E6%AD%A5%E5%99%A8%EF%BC%88AQS%EF%BC%89%E5%88%B0%E5%BA%95%E6%98%AF%E6%80%8E%E4%B9%88%E4%B8%80%E5%9B%9E%E4%BA%8B/t1%E9%87%8A%E6%94%BE%E5%90%8C%E6%AD%A5%E7%8A%B6%E6%80%81.png?raw=true)

假设t2被唤醒后成功的获取到了同步状态（即调用tryAcquire方法并成功将state设置为1），t2会将自己所在的Node设置为head节点，并将原head节点的next设置为null（有助于GC）

![t2重新获取同步状态](https://github.com/Lord-X/awesome-it-blog/blob/master/images/java/Java%E9%98%9F%E5%88%97%E5%90%8C%E6%AD%A5%E5%99%A8%EF%BC%88AQS%EF%BC%89%E5%88%B0%E5%BA%95%E6%98%AF%E6%80%8E%E4%B9%88%E4%B8%80%E5%9B%9E%E4%BA%8B/t2%E9%87%8D%E6%96%B0%E8%8E%B7%E5%8F%96%E5%90%8C%E6%AD%A5%E7%8A%B6%E6%80%81.png?raw=true)

t2执行完成，释放同步状态，将state设置为0，同时唤醒t3，使之再次具备竞争资格

![t2释放同步状态](https://github.com/Lord-X/awesome-it-blog/blob/master/images/java/Java%E9%98%9F%E5%88%97%E5%90%8C%E6%AD%A5%E5%99%A8%EF%BC%88AQS%EF%BC%89%E5%88%B0%E5%BA%95%E6%98%AF%E6%80%8E%E4%B9%88%E4%B8%80%E5%9B%9E%E4%BA%8B/t2%E9%87%8A%E6%94%BE%E5%90%8C%E6%AD%A5%E7%8A%B6%E6%80%81.png?raw=true)

假设t3成功获取同步状态，此时t3将自己所在的Node设置为head节点，并将之前的head节点的next设置为null（即将t2的next设置为null）

![t3重新获取同步状态](https://github.com/Lord-X/awesome-it-blog/blob/master/images/java/Java%E9%98%9F%E5%88%97%E5%90%8C%E6%AD%A5%E5%99%A8%EF%BC%88AQS%EF%BC%89%E5%88%B0%E5%BA%95%E6%98%AF%E6%80%8E%E4%B9%88%E4%B8%80%E5%9B%9E%E4%BA%8B/t3%E9%87%8D%E6%96%B0%E8%8E%B7%E5%8F%96%E5%90%8C%E6%AD%A5%E7%8A%B6%E6%80%81.png?raw=true)

t3执行完毕，释放同步状态，将state设置为0。由于此时其waitStatus等于0，表示已经没有后继节点需要unpark，直接返回释放成功

![t3释放同步状态](https://github.com/Lord-X/awesome-it-blog/blob/master/images/java/Java%E9%98%9F%E5%88%97%E5%90%8C%E6%AD%A5%E5%99%A8%EF%BC%88AQS%EF%BC%89%E5%88%B0%E5%BA%95%E6%98%AF%E6%80%8E%E4%B9%88%E4%B8%80%E5%9B%9E%E4%BA%8B/t3%E9%87%8A%E6%94%BE%E8%B5%84%E6%BA%90.png?raw=true)

最后的t3节点并没有被清空，因为他可以用作下一次同步状态竞争的head节点。

#### 2.6 超时获取同步状态

tryAcquireNanos方法实现了这个功能。他与上面描述的获取同步状态的过程大致相同，只不过是加上了时间的判断。
也就是说，每次自旋获取同步状态时，先判断当前时间是否超过了指定的超时时间，如果超时直接返回获取失败。

下面来看下源码，tryAcquireNanos方法源码如下：

```java
public final boolean tryAcquireNanos(int arg, long nanosTimeout)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();

    // 先尝试获取同步状态，如果失败，尝试超时获取
    return tryAcquire(arg) ||
        doAcquireNanos(arg, nanosTimeout);
}
```

可以发现，最终是doAcquireNanos方法实现的超时功能，这个方法中，大部分逻辑与上面的过程是一直的。
注释中说明了有区别的地方。

```java
private boolean doAcquireNanos(int arg, long nanosTimeout)
        throws InterruptedException {
    if (nanosTimeout <= 0L)
        return false;

    // 计算出超时那个时间点的时间戳
    final long deadline = System.nanoTime() + nanosTimeout;
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return true;
            }
            // 判断，如果超时，直接返回获取失败
            nanosTimeout = deadline - System.nanoTime();
            if (nanosTimeout <= 0L)
                return false;
            // 没超时的话，判断剩余时间是否大于1000纳秒，如果大于才park当前线程
            // 否则，不park，直接进入下一次自旋获取，因为这个时间足够小了，可能已经超出了一次系统调用的时间
            if (shouldParkAfterFailedAcquire(p, node) &&
                nanosTimeout > spinForTimeoutThreshold) // spinForTimeoutThreshold = 1000
                LockSupport.parkNanos(this, nanosTimeout);
            if (Thread.interrupted())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

### 3 参考

* Java并发编程的艺术 方腾飞，魏鹏，程晓明