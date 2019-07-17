LongAdder是JDK8添加到JUC中的。它是一个线程安全的、比Atomic*系工具性能更好的"计数器"。

它提供的方法主要有下面这些：

| 方法名  | 说明 |
| ------ | ------ |
| void add(long x) | 将当前的value加x。 |
| void increment() | 将当前的value加1。|
| void decrement() | 将当前的value减1。|
| long sum() | 返回当前值。特别注意，在没有并发更新value的情况下，sum会返回一个精确值，在存在并发的情况下，sum不保证返回精确值。原因在下文中会说明。|
| void reset() | 将value重置为0，可用于替代重新new一个LongAdder，但此方法只可以在没有并发更新的情况下使用。|
| long sumThenReset() | 获取当前value，并将value重置为0。|


### 0 LongAdder的类图

![LongAdder类图](http://image.feathers.top/image/LongAdder类图.png)

LongAdder本身没有成员变量，其值的变更实际上是由父类Striped64管理的。

Striped64通过两个成员变量来管理value，分别是base和cells，cells是一个数组，其元素是Striped64的内部类Cell的实现，Cell很简单，只记录一个value。

当LongAdder不存在并发访问的时候，会直接通过cas的方式更新base的值，存在并发访问时，会定位到某一个cell，修改cell的value。

最终，value = base + sum(cells)。

### 1 深挖add方法

add方法的作用是将当前的value值 + x。其实弄懂add方法后，increment方法和decrement自然就明白了，这两个方法里面调的其实就是add方法。

```java
/**
 * Equivalent to {@code add(1)}.
 */
public void increment() {
    add(1L);
}

/**
 * Equivalent to {@code add(-1)}.
 */
public void decrement() {
    add(-1L);
}
```

#### 1.1 add方法的实现

其源码如下：

```java
public void add(long x) {
    Cell[] as; long b, v; int m; Cell a;
    // cells和base即Striped64维护的实例变量
    if ((as = cells) != null || !casBase(b = base, b + x)) { // #1
        boolean uncontended = true;
        if (as == null || (m = as.length - 1) < 0 ||
            (a = as[getProbe() & m]) == null ||
            !(uncontended = a.cas(v = a.value, v + x)))
            longAccumulate(x, null, uncontended);
    }
}
```

初始状态下，cells==null，所以#1代码中，第一个条件一定是false，此时会通过casBase方法，以CAS的方式更新base值，且只有当cas失败时，才会走到if中。

啥时候会失败呢？存在竞争的时候。由此可以看出，当不存在竞争的时候，LongAdder是通过累加base值实现value的更新的。

当存在竞争的时候，LongAdder会尝试定位到其中的一个Cell，通过更新这个Cell的值来维护整体的value。这个过程如下：

![LongAdder.add](http://image.feathers.top/image/LongAdder.add.png)

longAccumulate方法在Striped64中定义。下面是当cells==null时的处理（暂时省略其他无关代码，能说明问题即可）：

```java
final void longAccumulate(long x, LongBinaryOperator fn, boolean wasUncontended) {
    int h;
    if ((h = getProbe()) == 0) {
        ThreadLocalRandom.current(); // force initialization
        h = getProbe();
        wasUncontended = true;
    }
    boolean collide = false;                // True if last slot nonempty
    for (;;) {
        Cell[] as; Cell a; int n; long v;
        if ((as = cells) != null && (n = as.length) > 0) {
            // ... 省略 ...
        }
        // cells == null 时的处理
        else if (cellsBusy == 0 && cells == as && casCellsBusy()) {
            boolean init = false;
            try {                           // Initialize table
                if (cells == as) {
                    Cell[] rs = new Cell[2];
                    rs[h & 1] = new Cell(x);
                    cells = rs;
                    init = true;
                }
            } finally {
                cellsBusy = 0;
            }
            if (init)
                break;
        }
        else if (casBase(v = base, ((fn == null) ? v + x :
                                    fn.applyAsLong(v, x))))
            break;                          // Fall back on using base
    }
}
```

cells==null时的处理在注释中已经标出。cellsBusy是用来控制并发修改cells的变量，初始值为0，通过casCellsBusy将cellsBusy的值从0更新为1。
然后，new了一个初始大小为2的Cell数组，new了一个Cell并将add的值作为初始值，然后计算一个cells的index，将这个Cell赋值进去。
最后，更新实例变量cells，并将cellsBusy重置为0。

cells!=null时，处理如下：

```java
final void longAccumulate(long x, LongBinaryOperator fn, boolean wasUncontended) {
    int h;
    if ((h = getProbe()) == 0) {
        ThreadLocalRandom.current(); // force initialization
        h = getProbe();
        wasUncontended = true;
    }
    boolean collide = false;                // True if last slot nonempty
    for (;;) {
        Cell[] as; Cell a; int n; long v;
        if ((as = cells) != null && (n = as.length) > 0) {
            // 从cells中定位一个Cell，如果是null，就new一个Cell，并将x作为初始值
            if ((a = as[(n - 1) & h]) == null) {
                if (cellsBusy == 0) {       // Try to attach new Cell
                    Cell r = new Cell(x);   // Optimistically create
                    if (cellsBusy == 0 && casCellsBusy()) {
                        boolean created = false;
                        try {               // Recheck under lock
                            Cell[] rs; int m, j;
                            if ((rs = cells) != null &&
                                (m = rs.length) > 0 &&
                                rs[j = (m - 1) & h] == null) {
                                rs[j] = r;
                                created = true;
                            }
                        } finally {
                            cellsBusy = 0;
                        }
                        if (created)
                            break;
                        continue;           // Slot is now non-empty
                    }
                }
                collide = false;
            }
            else if (!wasUncontended)       // CAS already known to fail
                wasUncontended = true;      // Continue after rehash
                
            // 如果定位到的Cell!=null，尝试通过cas的方式更新这个cell维护的value。
            // 如果更新成功，退出循环
            else if (a.cas(v = a.value, ((fn == null) ? v + x :
                                         fn.applyAsLong(v, x))))
                break;
            else if (n >= NCPU || cells != as)
                collide = false;            // At max size or stale
            else if (!collide)
                collide = true;
            // 如果依然更新失败，扩容cells为原来的两倍，并将原来cells中的元素复制到新的cells中。
            // PS: 通过cellsBusy控制并发
            else if (cellsBusy == 0 && casCellsBusy()) {
                try {
                    if (cells == as) {      // Expand table unless stale
                        Cell[] rs = new Cell[n << 1];
                        for (int i = 0; i < n; ++i)
                            rs[i] = as[i];
                        cells = rs;
                    }
                } finally {
                    cellsBusy = 0;
                }
                collide = false;
                continue;                   // Retry with expanded table
            }
            h = advanceProbe(h);
        }
        else if (cellsBusy == 0 && cells == as && casCellsBusy()) {
            // ... 省略 ...
        }
        // 上述过程都失败，尝试更新base的值。
        else if (casBase(v = base, ((fn == null) ? v + x :
                                    fn.applyAsLong(v, x))))
            break;                          // Fall back on using base
    }
}
```

上述代码的大致原理如下:

![LongAdder.add原理](http://image.feathers.top/image/LongAdder.add原理.png)


#### 1.2 它为啥比AtomicLong更加高效

以AtomicLong的getAndAdd方法做一下对比。下面是这个方法的源码：

```java
/**
 * Atomically adds the given value to the current value.
 *
 * @param delta the value to add
 * @return the previous value
 */
public final long getAndAdd(long delta) {
    return unsafe.getAndAddLong(this, valueOffset, delta);
}

public final long getAndAddLong(Object var1, long var2, long var4) {
    long var6;
    do {
        var6 = this.getLongVolatile(var1, var2);
    } while(!this.compareAndSwapLong(var1, var2, var6, var6 + var4));

    return var6;
}
```
AtomicLong的实现是通过无限循环cas的方式更新当前维护的值，可以想象，在并发足够大的情况下，cas的失败率会很高，这里循环次数剧增，造成CPU使用率飙高。

通过上文分析LongAdder.add(int x)的原理，LongAdder先尝试一次cas更新，如果失败会转而通过Cell[]的方式更新值，如果计算index的方式足够散列，那么在并发量大的情况下，多个线程定位到同一个cell的概率也就越低，这有点类似于分段锁的意思。

由此也可以分析出另一点，在并发量不大的情况下，二者的性能是没有多大差异的。

### 2 深挖sum方法

sum方法用于返回LongAdder当前的值。

#### 2.1 sum方法的实现

sum方法的实现很简单，其实就是 base + sum(cells)。源码如下：

```java
public long sum() {
    Cell[] as = cells; Cell a;
    long sum = base;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    return sum;
}
```

#### 2.2 为啥在并发情况下sum的值不精确

由上述源码可以发现，sum执行时，并没有限制对base和cells的更新。也就是说，对LongAdder的最后一次更新not happens-before 最近的一次读取。

首先，最终返回的sum局部变量，初始被复制为base，而最终返回时，很可能base已经被更新了，而此时局部变量sum不会更新，造成不一致。

其次，这里对cell的读取也无法保证是最后一次写入的值。

所以，sum方法在没有并发的情况下，可以获得正确的结果。

### 3 深挖reset方法

用于初始化base和cells数组，将值重置为0。

#### 3.1 reset方法的实现

源码如下：

```java
public void reset() {
    Cell[] as = cells; Cell a;
    base = 0L;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                a.value = 0L;
        }
    }
}
```

其实就是将base和cells数组挨个设置成0。

#### 3.2 为啥不可以在并发情况下使用

因为reset方法并不是原子操作，它先将base重置成0，假设此时CPU切走执行sum，此时的sum就变成了减去base后的值。

也就是说，在并发量大的情况下，执行完此方法并不能保证其他线程看到的是重置后的结果。所以要慎用。


### 4 总结

在实际工作中，可根据LongAdder和AtomicLong的特点来使用这两个工具。
当需要在高并发下有较好的性能表现，且对值的精确度要求不高时，可以使用LongAdder（例如网站访问人数计数）。
当需要保证线程安全，可允许一些性能损耗，要求高精度时，可以使用AtomicLong。