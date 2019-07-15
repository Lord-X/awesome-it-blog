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


#### 1.2 它为啥比AtomicLong更加高效



### 2 深挖sum方法

#### 2.1 sum方法的实现

#### 2.2 为啥在并发情况下sum的值不精确


### 3 深挖reset方法

#### 3.1 reset方法的实现

#### 3.2 为啥不可以在并发情况下使用