## 开源推荐
推荐一款一站式性能监控工具（开源项目）

Pepper-Metrics是跟一位同事一起开发的开源组件，主要功能是通过比较轻量的方式与常用开源组件（jedis/mybatis/motan/dubbo/servlet）集成，收集并计算metrics，并支持输出到日志及转换成多种时序数据库兼容数据格式，配套的grafana dashboard友好的进行展示。项目当中原理文档齐全，且全部基于SPI设计的可扩展式架构，方便的开发新插件。另有一个基于docker-compose的独立demo项目可以快速启动一套demo示例查看效果[https://github.com/zrbcool/pepper-metrics-demo](https://github.com/zrbcool/pepper-metrics-demo)。如果大家觉得有用的话，麻烦给个star，也欢迎大家参与开发，谢谢：）

---

## 进入正题...

GC的出现解放了程序员需要手动回收内存的苦恼，但我们也是要了解GC的，知己知彼，百战不殆嘛。
常见的GC回收算法主要包括引用计数算法、标记清除算法、复制算法、标记压缩算法、分代算法以及分区算法。
今天来聊聊引用计数算法。

### 1 原理
顾名思义，此种算法会在每一个对象上记录这个对象被引用的次数，只要有任何一个对象引用了次对象，这个对象的计数器就+1，取消对这个对象的引用时，计数器就-1。任何一个时刻，如果该对象的计数器为0，那么这个对象就是可以回收的。

打个比方：

```java
public static void method() {
    A a = new A();
}

public static void main(String[] args) {
    method();
}
```

main函数调用method方法，method方法中new了一个A的对象，赋值给局部变量a，此时堆内存中的对象A的实例的计数器就会+1。当方法结束时，局部变量会随之销毁，堆内存中的对象的计数器就会-1。

### 2 存在的问题
该算法存在两个问题：
* 无法处理循环引用的情况。
* 从上述的原理可知，堆内对象的每一次引用赋值和每一次引用清除，都伴随着加减法的操作，会带来一定的性能开销。

所以Java没有使用这种算法来实现GC。

下面来解释一下第一个问题，循环引用的情况。

即对象A引用对象B，对象B引用对象A。

考虑如下代码：

```java
class A {
    private B b;
    public void setB(B b) {
        this.b = b;
    }
}

class B {
    private A a = new A();
    public void setA(A a) {
        this.a = a;
    }
}

public void method() {
    A a = new A();
    B b = new B();
    a.setB(b);
    b.setA(a);
}
```

其内存图示如下

![GC引用计数法-循环引用图示](http://feathers.zrbcool.top/image/GC%E5%BC%95%E7%94%A8%E8%AE%A1%E6%95%B0%E6%B3%95-%E5%BE%AA%E7%8E%AF%E5%BC%95%E7%94%A8%E5%9B%BE%E7%A4%BA.jpg)

method方法中，执行完两个set后，method方法结束，图中两条红线引用消失，可以看到，留下两个对象在堆内存中循环引用，但此时已经没有地方在用他们了，造成内存泄漏。两个对象就凌乱在风中不知所措了。