本文分析了双亲委派模型的实现原理，并通过代码示例说明了什么时候需要实现自己的类加载器以及如何实现自己的类加载器。

本文基于JDK8。

### 0 ClassLoader的作用
ClassLoader用于将class文件加载到JVM中。另外一个作用是确认每个类应该由哪个类加载器加载。

第二个作用也用于判断JVM运行时的两个类是否相等，影响的判断方法有equals()、isAssignableFrom()、isInstance()以及instanceof关键字，这一点在后文中会举例说明。

#### 0.1 何时出发类加载动作？
类加载的触发可以分为隐式加载和显示加载。

**隐式加载**

隐式加载包括以下几种情况：
* 遇到new、getstatic、putstatic、invokestatic这4条字节码指令时
* 对类进行反射调用时
* 当初始化一个类时，如果其父类还没有初始化，优先加载其父类并初始化
* 虚拟机启动时，需指定一个包含main函数的主类，优先加载并初始化这个主类

**显示加载**

显示加载包含以下几种情况：
* 通过ClassLoader的loadClass方法
* 通过Class.forName
* 通过ClassLoader的findClass方法

#### 0.2 被加载的类存放在哪里
JDK8之前会加载到内存中的方法区。
从JDK8到现在为止，会加载到元数据区。

### 1 都有哪些ClassLoader
整个JVM平台提供三类ClassLoader。
#### 1.1 Bootstrap ClassLoader
加载JVM自身工作需要的类，它由JVM自己实现。它会加载$JAVA_HOME/jre/lib下的文件

#### 1.2 ExtClassLoader
它是JVM的一部分，由sun.misc.Launcher$ExtClassLoader实现，他会加载$JAVA_HOME/jre/lib/ext目录中的文件（或由System.getProperty("java.ext.dirs")所指定的文件）。

#### 1.3 AppClassLoader
应用类加载器，我们工作中接触最多的也是这个类加载器，它由sun.misc.Launcher$AppClassLoader实现。它加载由System.getProperty("java.class.path")指定目录下的文件，也就是我们通常说的classpath路径。

### 2 双亲委派模型
#### 2.1 双亲委派模型原理
从JDK1.2之后，类加载器引入了双亲委派模型，其模型图如下：

![ClassLoaderParentMod](http://feathers.zrbcool.top/image/Classloader.jpg)

其中，两个用户自定义类加载器的父加载器是AppClassLoader，AppClassLoader的父加载器是ExtClassLoader，ExtClassLoader是没有父类加载器的，在代码中，ExtClassLoader的父类加载器为null。BootstrapClassLoader也并没有子类，因为他完全由JVM实现。

**双亲委派模型的原理是**：当一个类加载器接收到类加载请求时，首先会请求其父类加载器加载，每一层都是如此，当父类加载器无法找到这个类时（根据类的全限定名称），子类加载器才会尝试自己去加载。

为了说明这个继承关系，我这里实现了一个自己的类加载器，名为TestClassLoader，在类加载器中，用parent字段来表示当前加载器的父类加载器，其定义如下：

```
public abstract class ClassLoader {
...
    // The parent class loader for delegation
    // Note: VM hardcoded the offset of this field, thus all new fields
    // must be added *after* it.
    private final ClassLoader parent;
...
}
```
然后通过debug来看一下这个结构，如下图

![ClassLoader委派模型关系图](http://feathers.zrbcool.top/image/classloader%E7%BB%A7%E6%89%BF%E5%85%B3%E7%B3%BB%E5%9B%BE.jpg)

这里的第一个红框是我自己定义的类加载器，对应上图的最下层部分；第二个框是自定义类加载器的父类加载器，可以看到是AppClassLoader；第三个框是AppClassLoader的父类加载器，是ExtClassLaoder；第四个框是ExtClassLoader的父类加载器，是null。

OK，这里先有个直观的印象，后面的实现原理中会详细介绍。

#### 2.2 此模型解决的问题
为什么要使用双亲委派模型呢？它可以解决什么问题呢？

双亲委派模型是JDK1.2之后引入的。根据双亲委派模型原理，可以试想，没有双亲委派模型时，如果用户自己写了一个全限定名为java.lang.Object的类，并用自己的类加载器去加载，同时BootstrapClassLoader加载了rt.jar包中的JDK本身的java.lang.Object，这样内存中就存在两份Object类了，此时就会出现很多问题，例如根据全限定名无法定位到具体的类。

有了双亲委派模型后，所有的类加载操作都会优先委派给父类加载器，这样一来，即使用户自定义了一个java.lang.Object，但由于BootstrapClassLoader已经检测到自己加载了这个类，用户自定义的类加载器就不会再重复加载了。

所以，双亲委派模型能够保证类在内存中的唯一性。

#### 2.3 双亲委派模型实现原理

下面从源码的角度看一下双亲委派模型的实现。

JVM在加载一个class时会先调用classloader的loadClassInternal方法，该方法源码如下

```
// This method is invoked by the virtual machine to load a class.
private Class<?> loadClassInternal(String name)
    throws ClassNotFoundException
{
    // For backward compatibility, explicitly lock on 'this' when
    // the current class loader is not parallel capable.
    if (parallelLockMap == null) {
        synchronized (this) {
             return loadClass(name);
        }
    } else {
        return loadClass(name);
    }
}
```

该方法里面做的事儿就是调用了loadClass方法，loadClass方法的实现如下
```
protected Class<?> loadClass(String name, boolean resolve)
    throws ClassNotFoundException
{
    synchronized (getClassLoadingLock(name)) {
        // First, check if the class has already been loaded
        // 先查看这个类是否已经被自己加载了
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            long t0 = System.nanoTime();
            try {
                // 如果有父类加载器，先委派给父类加载器来加载
                if (parent != null) {
                    c = parent.loadClass(name, false);
                } else {
                    // 如果父类加载器为null，说明ExtClassLoader也没有找到目标类，则调用BootstrapClassLoader来查找
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
                // ClassNotFoundException thrown if class not found
                // from the non-null parent class loader
            }
            // 如果都没有找到，调用findClass方法，尝试自己加载这个类
            if (c == null) {
                // If still not found, then invoke findClass in order
                // to find the class.
                long t1 = System.nanoTime();
                c = findClass(name);

                // this is the defining class loader; record the stats
                sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                sun.misc.PerfCounter.getFindClasses().increment();
            }
        }
        if (resolve) {
            resolveClass(c);
        }
        return c;
    }
}
```

源码中已经给出了几个关键步骤的说明。

代码中调用BootstrapClassLoader的地方实际是调用的native方法。

由此可见，双亲委派模型实现的核心就是这个loadClass方法。

### 3 实现自己的类加载器

#### 3.1 为什么要实现自己的类加载器
回答这个问题首先要思考类加载器有什么作用（粗体标出）。

##### 3.1.1 类加载器的作用

类加载器有啥作用呢？我们再回到上面的源码。

从上文我们知道JVM通过loadClass方法来查找类，所以，**他的第一个作用也是最重要的：在指定的路径下查找class文件（各个类加载器的扫描路径在上文已经给出）。**

然后，当父类加载器都说没有加载过目标类时，他会尝试自己加载目标类，这就调用了findClass方法，可以看一下findClass方法的定义：
```
protected Class<?> findClass(String name) throws ClassNotFoundException {
    throw new ClassNotFoundException(name);
}
```
可以发现他要求返回一个Class对象实例，这里我通过一个实现类sun.rmi.rmic.iiop.ClassPathLoader来说明一下findClass都干了什么。
```
protected Class findClass(String var1) throws ClassNotFoundException {
    // 从指定路径加载指定名称的class的字节流
    byte[] var2 = this.loadClassData(var1);
    // 通过ClassLoader的defineClass来创建class对象实例
    return this.defineClass(var1, var2, 0, var2.length);
}
```
他做的事情在注释中已经给出，可以看到，最终是通过defineClass方法来实例化class对象的。

另外可以发现，class文件字节的获取和处理我们是可以控制的。**所以，第二个作用：我们可以在字节流解析这一步做一些自定义的处理。** 例如，加解密。

接下来，看似还有个defineClass可以让我们来做点儿什么，ClassLoader的实现如下：
```
protected final Class<?> defineClass(String name, byte[] b, int off, int len)
    throws ClassFormatError
{
    return defineClass(name, b, off, len, null);
}
```
被final掉了，没办法覆写，所以这里看似不能做什么事儿了。

**小结一下：**

* 通过loadClass在指定的路径下查找文件。
* 通过findClass方法解析class字节流，并实例化class对象。

##### 3.1.2 什么时候需要自己实现类加载器
当JDK提供的类加载器实现无法满足我们的需求时，才需要自己实现类加载器。

现有应用场景：OSGi、代码热部署等领域。

另外，根据上述类加载器的作用，可能有以下几个场景需要自己实现类加载器
* 当需要在自定义的目录中查找class文件时（或网络获取）
* class被类加载器加载前的加解密（代码加密领域）


#### 3.2 如何实现自己的类加载器
接下来，实现一个在自定义class类路径中查找并加载class的自定义类加载器。
```
package com.lordx.sprintbootdemo.classloader;

import java.io.ByteArrayOutputStream;
import java.io.File;
import java.io.FileInputStream;
import java.io.InputStream;

/**
 * 自定义ClassLoader
 * 功能：可自定义class文件的扫描路径
 * @author zhiminxu 
 */
// 继承ClassLoader，获取基础功能
public class TestClassLoader extends ClassLoader {

    // 自定义的class扫描路径
    private String classPath;

    public TestClassLoader(String classPath) {
        this.classPath = classPath;
    }

    // 覆写ClassLoader的findClass方法
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        // getDate方法会根据自定义的路径扫描class，并返回class的字节
        byte[] classData = getDate(name);
        if (classData == null) {
            throw new ClassNotFoundException();
        } else {
            // 生成class实例
            return defineClass(name, classData, 0, classData.length);
        }
    }


    private byte[] getDate(String name) {
        // 拼接目标class文件路径
        String path = classPath + File.separatorChar + name.replace('.', File.separatorChar) + ".class";
        try {
            InputStream is = new FileInputStream(path);
            ByteArrayOutputStream stream = new ByteArrayOutputStream();
            byte[] buffer = new byte[2048];
            int num = 0;
            while ((num = is.read(buffer)) != -1) {
                stream.write(buffer, 0 ,num);
            }
            return stream.toByteArray();
        } catch (Exception e) {
            e.printStackTrace();
        }
        return null;
    }
}
```
使用自定义的类加载器
```
package com.lordx.sprintbootdemo.classloader;

public class MyClassLoader {
    public static void main(String[] args) throws ClassNotFoundException {
        // 自定义class类路径
        String classPath = "/Users/zhiminxu/developer/classloader";
        // 自定义的类加载器实现：TestClassLoader
        TestClassLoader testClassLoader = new TestClassLoader(classPath);
        // 通过自定义类加载器加载
        Class<?> object = testClassLoader.loadClass("ClassLoaderTest");
        // 这里的打印应该是我们自定义的类加载器：TestClassLoader
        System.out.println(object.getClassLoader());
    }
}
```

实验中的ClassLoaderTest类就是一个简单的定义了两个field的Class。如下图所示

![classloadertest](http://feathers.zrbcool.top/image/classloadertest.jpg)

最终的打印结果如下

![Result](http://feathers.zrbcool.top/image/classloaderresult.jpg)

PS: 实验类（ClassLoaderTest）最好不要放在IDE的工程目录内，因为IDE在run的时候会先将工程中的所有类都加载到内存，这样一来这个类就不是自定义类加载器加载的了，而是AppClassLoader加载的。

#### 3.3 类加载器对“相等”判断的影响
##### 3.3.1 对Object.equals()的影响
还是上面那个自定义类加载器

修改MyClassLoader代码
```
package com.lordx.sprintbootdemo.classloader;

public class MyClassLoader {
    public static void main(String[] args) throws ClassNotFoundException {
        // 自定义class类路径
        String classPath = "/Users/zhiminxu/developer/classloader";
        // 自定义的类加载器实现：TestClassLoader
        TestClassLoader testClassLoader = new TestClassLoader(classPath);
        // 通过自定义类加载器加载
        Class<?> object1 = testClassLoader.loadClass("ClassLoaderTest");
        Class<?> object2 = testClassLoader.loadClass("ClassLoaderTest");
        // object1和object2使用同一个类加载器加载时
        System.out.println(object1.equals(object2));

        // 新定义一个类加载器
        TestClassLoader testClassLoader2 = new TestClassLoader(classPath);
        Class<?> object3 = testClassLoader2.loadClass("ClassLoaderTest");
        // object1和object3使用不同类加载器加载时
        System.out.println(object1.equals(object3));
    }
}

```
打印结果：

true

false

equals方法默认比较的是内存地址，由此可见，不同的类加载器实例都有自己的内存空间，即使类的全限定名相同，但类加载器不同也是不行的。

所以，内存中两个类equals为true的条件是用同一个类加载器实例加载的全限定名相同的两个类实例。

##### 3.3.2 对instanceof的影响
修改TestClassLoader，增加main方法来实验，修改后的TestClassLoader如下：
```
import java.io.ByteArrayOutputStream;
import java.io.File;
import java.io.FileInputStream;
import java.io.InputStream;

/**
 * 自定义ClassLoader
 * 功能：可自定义class文件的扫描路径
 * @author zhiminxu
 */
// 继承ClassLoader，获取基础功能
public class TestClassLoader extends ClassLoader {

    public static void main(String[] args) throws ClassNotFoundException {
        TestClassLoader testClassLoader = new TestClassLoader("/Users/zhiminxu/developer/classloader");
        Object obj = testClassLoader.loadClass("ClassLoaderTest");
        // obj是testClassLoader加载的，ClassLoaderTest是AppClassLoader加载的，所以这里打印:false
        System.out.println(obj instanceof ClassLoaderTest);
    }

    // 自定义的class扫描路径
    private String classPath;

    public TestClassLoader(String classPath) {
        this.classPath = classPath;
    }

    // 覆写ClassLoader的findClass方法
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        // getDate方法会根据自定义的路径扫描class，并返回class的字节
        byte[] classData = getDate(name);
        if (classData == null) {
            throw new ClassNotFoundException();
        } else {
            // 生成class实例
            return defineClass(name, classData, 0, classData.length);
        }
    }


    private byte[] getDate(String name) {
        // 拼接目标class文件路径
        String path = classPath + File.separatorChar + name.replace('.', File.separatorChar) + ".class";
        try {
            InputStream is = new FileInputStream(path);
            ByteArrayOutputStream stream = new ByteArrayOutputStream();
            byte[] buffer = new byte[2048];
            int num = 0;
            while ((num = is.read(buffer)) != -1) {
                stream.write(buffer, 0 ,num);
            }
            return stream.toByteArray();
        } catch (Exception e) {
            e.printStackTrace();
        }
        return null;
    }
}
```
打印结果：

false

其他的还会影响Class的isAssignableFrom()方法和isInstance()方法，原因与上面的相同。

##### 3.3.3 另外的说明
不要尝试自定义java.lang包，并尝试用加载器去加载他们。像下面这样

![selfjavalang](http://feathers.zrbcool.top/image/selfclass.jpg)

这么干的话，会直接抛出一个异常

![error](http://feathers.zrbcool.top/image/javalangerror.jpg)

这个异常是在调用defineClass的校验过程抛出的，源码如下
```
// Note:  Checking logic in java.lang.invoke.MemberName.checkForTypeAlias
// relies on the fact that spoofing is impossible if a class has a name
// of the form "java.*"
if ((name != null) && name.startsWith("java.")) {
    throw new SecurityException
        ("Prohibited package name: " +
         name.substring(0, name.lastIndexOf('.')));
}
```

### 4 跟ClassLoader相关的几个异常
想知道如下几个异常在什么情况下抛出，其实只需要在ClassLoader中找到哪里会抛出他，然后在看下相关逻辑即可。
#### 4.1 ClassNotFoundException
这个异常，相信大家经常遇到。

那么，到底啥原因导致抛出这个异常呢？

看一下ClassLoader的源码，在JVM调用loadClassInternal的方法中，就会抛出这个异常。

其声明如下：
```
// This method is invoked by the virtual machine to load a class.
private Class<?> loadClassInternal(String name)
    throws ClassNotFoundException
{
    // For backward compatibility, explicitly lock on 'this' when
    // the current class loader is not parallel capable.
    if (parallelLockMap == null) {
        synchronized (this) {
             return loadClass(name);
        }
    } else {
        return loadClass(name);
    }
}
```
这里面loadClass方法会抛出这个异常，再来看loadClass方法
```
public Class<?> loadClass(String name) throws ClassNotFoundException {
    return loadClass(name, false);
}
```
调用了重写的loadClass方法
```
protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
{
    synchronized (getClassLoadingLock(name)) {
        // First, check if the class has already been loaded
        // 查看findLoadedClass的声明，没有抛出这个异常
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            long t0 = System.nanoTime();
            try {
                if (parent != null) {
                    // 这里会抛，但同样是调loadClass方法，无需关注
                    c = parent.loadClass(name, false);
                } else {
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
                // catch住没有抛，因为要在下面尝试自己获取class
                // ClassNotFoundException thrown if class not found
                // from the non-null parent class loader
            }

            if (c == null) {
                // If still not found, then invoke findClass in order
                // to find the class.
                long t1 = System.nanoTime();
                // 关键：这里会抛
                c = findClass(name);

                // this is the defining class loader; record the stats
                sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                sun.misc.PerfCounter.getFindClasses().increment();
            }
        }
        if (resolve) {
            resolveClass(c);
        }
        return c;
    }
}
```
再来看findClass方法
```
/**
 * Finds the class with the specified <a href="#name">binary name</a>.
 * This method should be overridden by class loader implementations that
 * follow the delegation model for loading classes, and will be invoked by
 * the {@link #loadClass <tt>loadClass</tt>} method after checking the
 * parent class loader for the requested class.  The default implementation
 * throws a <tt>ClassNotFoundException</tt>.
 *
 * @param  name
 *         The <a href="#name">binary name</a> of the class
 *
 * @return  The resulting <tt>Class</tt> object
 *
 * @throws  ClassNotFoundException
 *          If the class could not be found
 *
 * @since  1.2
 */
protected Class<?> findClass(String name) throws ClassNotFoundException {
    throw new ClassNotFoundException(name);
}
```
果然这里抛出的，注释中抛出这个异常的原因是说：当这个class无法被找到时抛出。

这也就是为什么在上面自定义的类加载器中，覆写findClass方法时，如果没有找到class要抛出这个异常的原因。

**至此，这个异常抛出的原因就明确了：在双亲委派模型的所有相关类加载器中，目标类在每个类加载器的扫描路径中都不存在时，会抛出这个异常。**

#### 4.2 NoClassDefFoundError
这也是个经常会碰到的异常，而且不熟悉的同学可能经常搞不清楚什么时候抛出ClassNotFoundException，什么时候抛出NoClassDefFoundError。

我们还是到ClassLoader中搜一下这个异常，可以发现在defineClass方法中可能抛出这个异常，defineClass方法源码如下：
```
/**
 * ... 忽略注释和参数以及返回值的说明，直接看异常声明
 * 
 * @throws  ClassFormatError
 *          If the data did not contain a valid class
 *
 * 在这里。由于NoClassDefFoundError是Error下的，所以不用显示throws
 * @throws  NoClassDefFoundError
 *          If <tt>name</tt> is not equal to the <a href="#name">binary
 *          name</a> of the class specified by <tt>b</tt>
 *
 * @throws  IndexOutOfBoundsException
 *          If either <tt>off</tt> or <tt>len</tt> is negative, or if
 *          <tt>off+len</tt> is greater than <tt>b.length</tt>.
 *
 * @throws  SecurityException
 *          If an attempt is made to add this class to a package that
 *          contains classes that were signed by a different set of
 *          certificates than this class, or if <tt>name</tt> begins with
 *          "<tt>java.</tt>".
 */
protected final Class<?> defineClass(String name, byte[] b, int off, int len,
                                     ProtectionDomain protectionDomain)
    throws ClassFormatError
{
    protectionDomain = preDefineClass(name, protectionDomain);
    String source = defineClassSourceLocation(protectionDomain);
    Class<?> c = defineClass1(name, b, off, len, protectionDomain, source);
    postDefineClass(c, protectionDomain);
    return c;
}
```
继续追踪里面的方法，可以发现是preDefineClass方法抛出的，这个方法源码如下：
```
/* Determine protection domain, and check that:
    - not define java.* class,
    - signer of this class matches signers for the rest of the classes in
      package.
*/
private ProtectionDomain preDefineClass(String name,
                                        ProtectionDomain pd)
{
    // 这里显示抛出，可发现是在目标类名校验不通过时抛出的
    if (!checkName(name))
        throw new NoClassDefFoundError("IllegalName: " + name);

    // Note:  Checking logic in java.lang.invoke.MemberName.checkForTypeAlias
    // relies on the fact that spoofing is impossible if a class has a name
    // of the form "java.*"
    if ((name != null) && name.startsWith("java.")) {
        throw new SecurityException
            ("Prohibited package name: " +
             name.substring(0, name.lastIndexOf('.')));
    }
    if (pd == null) {
        pd = defaultDomain;
    }

    if (name != null) checkCerts(name, pd.getCodeSource());

    return pd;
}
```
这个校验的源码如下
```
// true if the name is null or has the potential to be a valid binary name
private boolean checkName(String name) {
    if ((name == null) || (name.length() == 0))
        return true;
    if ((name.indexOf('/') != -1)
        || (!VM.allowArraySyntax() && (name.charAt(0) == '[')))
        return false;
    return true;
}
```
**所以，这个异常抛出的原因是：类名校验未通过。**

但这个异常其实不止在ClassLoader中抛出，其他地方，例如框架中，web容器中都有可能抛出，还要具体问题具体分析。


### 5 总结
本文先介绍了ClassLoader的作用：主要用于从指定路径查找class并加载到内存，另外是判断两个类是否相等。

后面介绍了双亲委派模型，以及其实现原理，JDK中主要是在ClassLoader的loadClass方法中实现双亲委派模型的。

然后代码示例说明了如何实现自己的类加载器，以及类加载器的使用场景。

最后说明了在工作中常遇到的两个与类加载器相关的异常。

