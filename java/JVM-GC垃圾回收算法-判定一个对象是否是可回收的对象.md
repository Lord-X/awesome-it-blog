GC的出现解放了程序员需要手动回收内存的苦恼，但我们也是要了解GC的，知己知彼，百战不殆嘛。

常见的GC回收算法主要包括引用计数算法、可达性分析法、标记清除算法、复制算法、标记压缩算法、分代算法以及分区算法。

其中，引用计数法和可达性分析法用于判定一个对象是否可以回收，其他的算法为具体执行GC时的算法。

今天来聊聊可达性分析法，并说明一下什么样的对象才是真正可以被回收的。

在介绍引用计数法的时候，我们提到了此种算法的循环引用的缺陷，所以Java没有使用此种算法。

那Java使用的是啥算法来标记一个对象是否是垃圾对象呢？

Java是通过判断一个对象是否可触及，以及一个对象的引用类型（强引用、软引用、弱引用、虚引用）来决定是否回收这个对象。

本文将根据上述，分为两部分进行介绍。

最后会简单介绍一下GC回收过程中保证数据一致性的方法：Stop the World

### 1 如何判断一个对象是否可触及？
判断是否可触及包含两个要素：
* 通过可达性分析该对象到GC Root不可达，如果不可达会进行第一次标记。
* 已经丧失"自救"机会，如果没有自救机会，会进行第二次标记，此时该对象可回收。

#### 1.1 可达性分析
可达性分析法定义了一系列称为"GC Roots"的对象作为起始点，从这个起点开始向下搜索，每一条可达路径称为引用链，当一个对象没有任意一条引用链可以到达"GC Roots"时，那么就对这个对象进行第一次"可回收"标记。

**那么什么是GC Root呢？**

可以理解为由堆外指向堆内的引用。

**那么都有哪些对象可以作为GC Roots呢？**
包括如下几种
* 代码中某一方法中的局部变量
* 类变量（静态变量）
* 常量
* 本地方法栈中引用的对象
* 已启动且未停止的线程

下面以一段代码来简单说明一下前三类

```java
class Test {
    private static A a = new A(); // 静态变量
    public static final String CONTANT = "I am a string"; // 常量

    public static void main(String[] args) {
        A innerA = new A(); // 局部变量
    }
}

class A {
    ...
}
```

这段代码的运行时内存图示如下：

![可达性分析图1](http://feathers.zrbcool.top/image/%E5%8F%AF%E8%BE%BE%E6%80%A7%E5%88%86%E6%9E%901.jpg)

首先，类加载器加载Test类，会初始化静态变量a，将常量引用指向常量池中的字符串，完成Test类的加载；

然后，main方法执行，main方法会入虚拟机方法栈，执行main方法会在堆中创建A的对象，并赋值给局部变量innerA。

此时GC Roots状态如下：

![可达性分析图2](http://feathers.zrbcool.top/image/%E5%8F%AF%E8%BE%BE%E6%80%A7%E5%88%86%E6%9E%902.jpg)

当main方法执行完出栈后，变为：

![可达性分析图3](http://feathers.zrbcool.top/image/%E5%8F%AF%E8%BE%BE%E6%80%A7%E5%88%86%E6%9E%903.jpg)

第三个对象已经没有引用链可达GC Root，此时，第三个对象被第一次标记。

#### 1.2 对象的"自救"
一个被可达性分析标记为可回收的对象，是有机会进行自救的。前提是：覆写了Object的finalize()方法，且GC还没有执行该对象的finalize()方法。

先来看一下finalize方法的定义

```java
/**
* Called by the garbage collector on an object when garbage collection
* determines that there are no more references to the object.
* A subclass overrides the {@code finalize} method to dispose of
* system resources or to perform other cleanup.
* <p>
* The general contract of {@code finalize} is that it is invoked
* if and when the Java&trade; virtual
* machine has determined that there is no longer any
* means by which this object can be accessed by any thread that has
* not yet died, except as a result of an action taken by the
* finalization of some other object or class which is ready to be
* finalized. The {@code finalize} method may take any action, including
* making this object available again to other threads; the usual purpose
* of {@code finalize}, however, is to perform cleanup actions before
* the object is irrevocably discarded. For example, the finalize method
* for an object that represents an input/output connection might perform
* explicit I/O transactions to break the connection before the object is
* permanently discarded.
* <p>
* The {@code finalize} method of class {@code Object} performs no
* special action; it simply returns normally. Subclasses of
* {@code Object} may override this definition.
* <p>
* The Java programming language does not guarantee which thread will
* invoke the {@code finalize} method for any given object. It is
* guaranteed, however, that the thread that invokes finalize will not
* be holding any user-visible synchronization locks when finalize is
* invoked. If an uncaught exception is thrown by the finalize method,
* the exception is ignored and finalization of that object terminates.
* <p>
* After the {@code finalize} method has been invoked for an object, no
* further action is taken until the Java virtual machine has again
* determined that there is no longer any means by which this object can
* be accessed by any thread that has not yet died, including possible
* actions by other objects or classes which are ready to be finalized,
* at which point the object may be discarded.
* <p>
* The {@code finalize} method is never invoked more than once by a Java
* virtual machine for any given object.
* <p>
* Any exception thrown by the {@code finalize} method causes
* the finalization of this object to be halted, but is otherwise
* ignored.
*
* @throws Throwable the {@code Exception} raised by this method
* @see java.lang.ref.WeakReference
* @see java.lang.ref.PhantomReference
* @jls 12.6 Finalization of Class Instances
*/
protected void finalize() throws Throwable { }
```

大致翻译一下前两段：当GC判定某一对象不再通过任一形式被引用时，GC会调用该对象的finalize方法。方法执行时，可以进行任何操作，包括将这个对象再次赋值给某一变量引用，但其主要目的还是做一些对象的清除操作。

其实在finalize方法中，只要将这个对象的引用(this)再次赋值给某一变量，这个对象就可以"自救"。

如果一个对象在finalize阶段也没有完成自救，那么就真的要被回收了。

**下面演示一个"自救"的例子：**

```java
public class SaveMe {

    public static SaveMe saveMe;

    public static void main(String[] args) throws InterruptedException {
        saveMe = new SaveMe();
        saveMe = null; // 取消引用，经过可达性分析，上面new出来的对象不再可达GC Root
        System.gc(); // 第一次GC，会执行finalize方法
        Thread.sleep(1000);
        if (saveMe == null) {
            System.out.println("对象为null");
        } else {
            System.out.println("对象不为null");
        }
        // 经过上面的过程，对象已经自救了，这里再次将其引用置空
        saveMe = null;
        System.gc(); // 不会再执行finalize方法，没有机会自救了
        Thread.sleep(1000);
        if (saveMe == null) {
            System.out.println("对象为null");
        } else {
            System.out.println("对象不为null");
        }
    }

    // finalize方法全局只会执行一次
    @Override
    protected void finalize() throws Throwable {
        super.finalize();
        saveMe = this; // 进行自救
    }
}
```

上述代码很简明，可根据注释理解。代码执行结果如下：

![可达性分析图4](http://feathers.zrbcool.top/image/%E5%8F%AF%E8%BE%BE%E6%80%A7%E5%88%86%E6%9E%904.jpg)

### 2 不同引用类型的回收
Java中有四种引用类型，引用强度由强到弱：强引用、软引用、弱引用、虚引用。针对不同的引用类型，GC的回收策略不同。

#### 2.1 强引用
通过关键字new的对象就是强引用对象，强引用指向的对象任何时候都不会被回收，宁愿OOM也不会回收。

#### 2.2 软引用
如果一个对象持有软引用，那么当JVM堆空间不足时，会被回收。

一个类的软引用可以通过java.lang.ref.SoftReference持有。

#### 2.3 弱引用
如果一个对象持有弱引用，那么在GC时，只要发现弱引用对象，就会被回收。

一个类的弱引用可以通过java.lang.ref.WeakReference持有。

#### 2.4 虚引用
几乎和没有一样，随时可以被回收。

通过PhantomReference持有。

### 3 Stop the World
问题的出现：如果程序一边执行，一边进行可达性分析的标记操作，那么有可能刚标记完一个对象，这个对象又再次被赋值给其他的引用。这样就有可能回收掉正在使用的对象。

解决这个问题的方式就是Stop the World（STW），STW会在所有线程到达一个安全点时，暂停掉所有应用线程的执行，然后开始专心的标记垃圾对象。这样就保证了数据的一致性，不会导致误回收。