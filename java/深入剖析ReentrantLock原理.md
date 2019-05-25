ReentrantLock，重入锁，是JDK5中添加在并发包下的一个高性能的工具。顾名思义，ReentrantLock支持同一个线程在未释放锁的情况下重复获取锁。

每一个东西的出现一定是有价值的。既然已经有了元老级的synchronized，而且synchronized也支持重入，为什么Doug Lea还要专门写一个ReentrantLock呢？

### 0 ReentrantLock与synchronized的比较

#### 0.1 性能上的比较

首先，ReentrantLock的性能要优于synchronized。下面通过两段代码比价一下。
首先是synchronized：

```java
public class LockDemo2 {
    private static final Object lock = new Object(); // 定义锁对象
    private static int count = 0; // 累加数
    public static void main(String[] args) throws InterruptedException {
        long start = System.currentTimeMillis();
        CountDownLatch cdl = new CountDownLatch(100);
        // 启动100个线程对count累加，每个线程累加1000000次
        // 调用add函数累加，通过synchronized保证多线程之间的同步
        for (int i=0;i<100;i++) {
            new Thread(() -> {
                for (int i1 = 0; i1 <1000000; i1++) {
                    add();
                }
                cdl.countDown();
            }).start();
        }
        cdl.await();
        System.out.println("Time cost: " + (System.currentTimeMillis() - start) + ", count = " + count);
    }

    private static void add() {
        synchronized (lock) {
            count++;
        }
    }
}
```

然后是ReentrantLock：

```java
public class LockDemo3 {
    private static Lock lock = new ReentrantLock(); // 重入锁
    private static int count = 0;
    public static void main(String[] args) throws InterruptedException {
        long start = System.currentTimeMillis();
        CountDownLatch cdl = new CountDownLatch(100);
        for (int i=0;i<100;i++) {
            new Thread(() -> {
                for (int i1 = 0; i1 <1000000; i1++) {
                    add();
                }
                cdl.countDown();
            }).start();
        }
        cdl.await();
        System.out.println("Time cost: " + (System.currentTimeMillis() - start) + ", count = " + count);
    }
    // 通过ReentrantLock保证线程之间的同步
    private static void add() {
        lock.lock();
        count++;
        lock.unlock();
    }
}
```

下面是运行多次的结果对比：

| |synchronized|ReentrantLock|
|----|----|----|
|第一次|4620 ms|3360 ms|
|第二次|4086 ms|3138 ms|
|第三次|4650 ms|3408 ms|

总体来看，ReentrantLock的平均性能要比synchronized好20%左右。

#### 0.2 获取锁公平性的比较

公平性是啥概念呢？如果是公平的获取锁，就是说多个线程之间获取锁的时候要排队，依次获取锁；如果是不公平的获取锁，就是说多个线程获取锁的时候一哄而上，谁抢到是谁的。

由于synchronized是基于monitor机制实现的，它只支持非公平锁；但ReentrantLock同时支持公平锁和非公平锁。

#### 0.3 综述

除了上文所述，ReentrantLock还有一些其他synchronized不具备的特性，这里来总结一下。

| |synchronized|ReentrantLock|
|----|----|----|
|性能|相对较差|优于synchronized 20%左右|
|公平性|只支持非公平锁|同时支持公平锁与非公平锁|
|尝试获取锁的支持|不支持，一旦到了同步块，且没有获取到锁，就阻塞在这里|支持，通过tryLock方法实现，可通过其返回值判断是否成功获取锁，所以即使获取锁失败也不会阻塞在这里|
|超时的获取锁|不支持，如果一直获取不到锁，就会一直等待下去|支持，通过tryLock(time, TimeUnit)方法实现，如果超时了还没获取锁，就放弃获取锁，不会一直阻塞下去|
|是否可响应中断|不支持，不可响应线程的interrupt信号|支持，通过lockInterruptibly方法实现，通过此方法获取锁之后，线程可响应interrupt信号，并抛出InterruptedException异常|
|等待条件的支持|支持，通过wait、notify、notifyAll来实现|支持，通过Conditon接口实现，支持多个Condition，比synchronized更加灵活|

### 1 可重入功能的实现原理

ReentrantLock的实现基于队列同步器（AbstractQueuedSynchronizer，后面简称AQS），关于AQS的实现原理，可以看笔者的另一篇文章：
[AQS的实现原理](https://github.com/Lord-X/awesome-it-blog/blob/master/java/Java%E9%98%9F%E5%88%97%E5%90%8C%E6%AD%A5%E5%99%A8%EF%BC%88AQS%EF%BC%89%E5%88%B0%E5%BA%95%E6%98%AF%E6%80%8E%E4%B9%88%E4%B8%80%E5%9B%9E%E4%BA%8B.md)

ReentrantLock的可重入功能基于AQS的同步状态：state。

其原理大致为：当某一线程获取锁后，将state值+1，并记录下当前持有锁的线程，再有线程来获取锁时，判断这个线程与持有锁的线程是否是同一个线程，如果是，将state值再+1，如果不是，阻塞线程。
当线程释放锁时，将state值-1，当state值减为0时，表示当前线程彻底释放了锁，然后将记录当前持有锁的线程的那个字段设置为null，并唤醒其他线程，使其重新竞争锁。

```java
// acquires的值是1
final boolean nonfairTryAcquire(int acquires) {
	// 获取当前线程
	final Thread current = Thread.currentThread();
	// 获取state的值
	int c = getState();
	// 如果state的值等于0，表示当前没有线程持有锁
	// 尝试将state的值改为1，如果修改成功，则成功获取锁，并设置当前线程为持有锁的线程，返回true
	if (c == 0) {
		if (compareAndSetState(0, acquires)) {
			setExclusiveOwnerThread(current);
			return true;
		}
	}
	// state的值不等于0，表示已经有其他线程持有锁
	// 判断当前线程是否等于持有锁的线程，如果等于，将state的值+1，并设置到state上，获取锁成功，返回true
	// 如果不是当前线程，获取锁失败，返回false
	else if (current == getExclusiveOwnerThread()) {
		int nextc = c + acquires;
		if (nextc < 0) // overflow
			throw new Error("Maximum lock count exceeded");
		setState(nextc);
		return true;
	}
	return false;
}
```

### 2 非公平锁的实现原理

ReentrantLock有两个构造函数：

```java
// 无参构造，默认使用非公平锁（NonfairSync）
public ReentrantLock() {
	sync = new NonfairSync();
}

// 通过fair参数指定使用公平锁（FairSync）还是非公平锁（NonfairSync）
public ReentrantLock(boolean fair) {
	sync = fair ? new FairSync() : new NonfairSync();
}
```

sync是ReentrantLock的成员变量，是其内部类Sync的实例。NonfairSync和FairSync都是Sync类的子类。可以参考如下类关系图：

![ReentrantLock类关系图](https://github.com/Lord-X/awesome-it-blog/blob/master/images/java/%E6%B7%B1%E5%85%A5%E5%89%96%E6%9E%90ReentrantLock%E5%8E%9F%E7%90%86/1_ReentrantLock%E7%B1%BB%E5%9B%BE.png)

Sync继承了AQS，所以他具备了AQS的功能。同样的，NonfairSync和FairSync都是AQS的子类。

当我们通过无参构造函数获取ReentrantLock实例后，默认用的就是非公平锁。

下面将通过如下场景描述非公平锁的实现原理：假设一个线程(t1)获取到了锁，其他很多没获取到锁的线程(others_t)加入到了AQS的同步队列中等待，当这个线程执行完，释放锁后，其他线程重新非公平的竞争锁。

先来描述一下获取锁的方法：

```java
final void lock() {
	// 线程t1成功的将state的值从0改为1，表示获取锁成功
	if (compareAndSetState(0, 1))
		setExclusiveOwnerThread(Thread.currentThread());
	else
	    // others_t线程们没有获取到锁
		acquire(1);
}
```

如果获取锁失败，会调用AQS的acquire方法

```java
public final void acquire(int arg) {
    // tryAcquire是个模板方法，在NonfairSync中实现，如果在tryAcquire方法中依然获取锁失败，会将当前线程加入同步队列中等待（addWaiter）
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

tryAcquire的实现如下，其实是调用了上面的nonfairTryAcquire方法

```java
protected final boolean tryAcquire(int acquires) {
    return nonfairTryAcquire(acquires);
}
```

OK，此时t1获取到了锁，others_t线程们都跑到同步队列里等着了。

某一时刻，t1自己的任务执行完成，调用了释放锁的方法（unlock）。

```java
public void unlock() {
    // 调用AQS的release方法释放资源
    sync.release(1);
}
```

```java
public final boolean release(int arg) {
    // tryRelease也是模板方法，在Sync中实现
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            // 成功释放锁后，唤醒同步队列中的下一个节点，使之可以重新竞争锁
            // 注意此时不会唤醒队列第一个节点之后的节点，这些节点此时还是无法竞争锁
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

```java
protected final boolean tryRelease(int releases) {
    // 将state的值-1，如果-1之后等于0，释放锁成功
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```

这时锁被释放了，被唤醒的线程和新来的线程重新竞争锁（不包含同步队列后面的那些线程）。

回到lock方法中，由于此时所有线程都能通过CAS来获取锁，并不能保证被唤醒的那个线程能竞争过新来的线程，所以是非公平的。这就是非公平锁的实现。

这个过程大概可以描述为下图这样子：

![非公平锁的竞争](https://github.com/Lord-X/awesome-it-blog/blob/master/images/java/%E6%B7%B1%E5%85%A5%E5%89%96%E6%9E%90ReentrantLock%E5%8E%9F%E7%90%86/2_%E9%9D%9E%E5%85%AC%E5%B9%B3%E9%94%81%E7%9A%84%E7%AB%9E%E4%BA%89.png)

### 3 公平锁的实现原理

公平锁与非公平锁的释放锁的逻辑是一样的，都是调用上述的unlock方法，最大区别在于获取锁的时候。

```java
static final class FairSync extends Sync {
    private static final long serialVersionUID = -3000897897090466540L;
    // 获取锁，与非公平锁的不同的地方在于，这里直接调用的AQS的acquire方法，没有先尝试获取锁
    // acquire又调用了下面的tryAcquire方法，核心在于这个方法
    final void lock() {
        acquire(1);
    }

    /**
     * 这个方法和nonfairTryAcquire方法只有一点不同，在标注为#1的地方
     * 多了一个判断hasQueuedPredecessors，这个方法是判断当前AQS的同步队列中是否还有等待的线程
     * 如果有，返回true，否则返回false。
     * 由此可知，当队列中没有等待的线程时，当前线程才能尝试通过CAS的方式获取锁。
     * 否则就让这个线程去队列后面排队。
     */
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            // #1
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
}
```

通过注释可知，在公平锁的机制下，任何线程想要获取锁，都要排队，不可能出现插队的情况。这就是公平锁的实现原理。

这个过程大概可以描述为下图这样子：

![公平锁的竞争](https://github.com/Lord-X/awesome-it-blog/blob/master/images/java/%E6%B7%B1%E5%85%A5%E5%89%96%E6%9E%90ReentrantLock%E5%8E%9F%E7%90%86/3_%E5%85%AC%E5%B9%B3%E9%94%81%E7%9A%84%E7%AB%9E%E4%BA%89.png)








