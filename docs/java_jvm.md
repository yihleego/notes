# JVM

## 内存模型

### 程序计数器

### 虚拟机栈

### 本地方法栈

### 堆

### 方法区

### 直接内存

## GC

### CMS

### G1

### ZGC

## ClassLoader

众所周知，运行 Java 程序需要先把`.java`文件编译成`.class`文件，然后通过 JVM 加载字节码文件到内存运行，而`Classloader`所做的事情就是将`.class`文件加载到 JVM 中。

### BootstrapClassLoader

引导类加载器，又称启动类加载器，是最顶层的类加载器，主要用来加载核心类，如：`rt.jar`、`resources.jar`、`charsets.jar`等。执行`java`的命令中使用`-Xbootclasspath`指定需要附加加载类的路径。

需要注意的是，它不是`java.lang.ClassLoader`的子类，而是由 JVM 自身实现的该类`C++`实现，Java 程序无法访问该加载器。

### ExtClassloader

扩展类加载器，主要负责加载扩展类库，默认加载`%JAVA_HOME%/jre/lib/ext/`目下的所有jar包和类，或者由`java.ext.dirs`系统属性指定的jar包，
放入这个目录下的jar包对所有`AppClassloader`都是可见的，因为`ExtClassloader`是`AppClassloader`的父加载器。

### AppClassloader

系统类加载器，又称应用程序类加载器，它负责在 JVM 启动时加载`classpath`，或者`java.class.path`系统属性，或者操作系统`CLASSPATH`属性所指定的jar包和类路径。

调用`ClassLoader.getSystemClassLoader()`可以获取该类加载器，这里的`SystemClassLoader`即`AppClassloader`。
如果没有特别指定，则用户自定义的任何类加载器都将`AppClassloader`作为父加载器，这点通过`ClassLoader`类的无参构造函数可以知道如下：

```java
/**
 * Creates a new class loader using the <tt>ClassLoader</tt> returned by
 * the method {@link #getSystemClassLoader()
 * <tt>getSystemClassLoader()</tt>} as the parent class loader.
 *
 * <p> If there is a security manager, its {@link
 * SecurityManager#checkCreateClassLoader()
 * <tt>checkCreateClassLoader</tt>} method is invoked.  This may result in
 * a security exception.  </p>
 *
 * @throws  SecurityException
 *          If a security manager exists and its
 *          <tt>checkCreateClassLoader</tt> method doesn't allow creation
 *          of a new class loader.
 */
protected ClassLoader() {
    this(checkCreateClassLoader(), getSystemClassLoader());
}
```

用户自定义的无参加载器的父类加载器默认是`AppClassloader`，而`AppClassloader`的父加载器是`ExtClassloader`，通过下面代码可以验证：

```java
// Launcher$ExtClassLoader
ClassLoader.getSystemClassLoader().getParent();
```

一般我们都认为`ExtClassloader`的父类加载器是`BootstrapClassLoader`，但是其实他们之间根本是没有父子关系的，只是在`ExtClassloader`找不到要加载类时候会去委托`BootstrapClassLoader`去加载。

_注意，这里说的父子关系并不是继承关系，而是通过定义`private final ClassLoader parent`属性关联。_

### ContextClassLoader

线程上下文类加载器存在的目的主要是为了解决双亲委派机制下无法解决的问题。

如果你了解过 Tomcat 或写过集团中间件，那么你应该对线程上下文类加载器很熟悉，可以很多地方看到下面的结构的用法：

```java
ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
try {
    Thread.currentThread().setContextClassLoader(targetTccl);
    // do something 
} finally {
    Thread.currentThread().setContextClassLoader(classLoader);
}
```

我们知道Java默认的类加载机制是委托机制，但是有时这种加载顺序不能正常工作，通常发生在有些 JVM 核心代码必须动态加载由应用程序开发人员提供的资源时。

Java 提供了很多服务提供者接口（Service Provider Interface，SPI），允许第三方厂商为这些接口提供实现。
常见的 SPI 有 JDBC、JCE、JNDI、JAXP 和 JBI 等，这些 SPI 的实现代码很可能是作为 Java 应用所依赖的 jar 包被包含进来。
问题在于 SPI 接口是 Java 核心库的一部分，是由引导类加载器来加载的，而 SPI 实现类一般是由系统类加载器来加载的。引导类加载器是无法找到 SPI 的实现类的，因为它只加载 Java 的核心库。

线程上下文类加载器正好解决了这个问题。如果不做任何的设置，Java 应用的线程的上下文类加载器默认就是系统上下文类加载器。在 SPI 接口的代码中使用线程上下文类加载器，就可以成功的加载到 SPI 实现的类。

### UserClassLoader

用户自定义加载类继承`ClassLoader`重写`findClass`方法即可，无法被父类加载器加载的类最终会通过`findClass`方法被加载。如果想打破双亲委派模型则需要重写`loadClass`方法。

### 双亲委派

双亲委派机制，也就是子类加载器在加载一个类时候会让父类来加载，那么问题来了，为什么使用这种方式呢？因为这样可以避免重复加载，当父类加载器已经加载了该类的时候，就没有必要用子类加载器再加载一次。
考虑到安全因素，我们试想一下，如果不使用这种模式，那我们就可以随时使用自定义的`String`来动态替代核心`API`中定义的类型，这样会存在非常大的安全隐患，而双亲委派的机制就可以避免这种情况，因为`String`已经在启动时就被`BootstrapClassLoader`加载，
所以用户自定义的`ClassLoader`永远也无法加载一个自己写的`String`，除非`JDK`源码中`ClassLoader`搜索类的默认算法。

```java
protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
    synchronized (getClassLoadingLock(name)) {
        // 判断类是否已被加载，如果已加载则直接返回
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            long t0 = System.nanoTime();
            try {
                if (parent != null) {
                    // 委托父加载器进行加载类
                    c = parent.loadClass(name, false);
                } else {
                    // 委托引导类加载器进行加载类
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
                // ClassNotFoundException thrown if class not found
                // from the non-null parent class loader
            }

            if (c == null) {
                // 如果依然没找到则调用 findClass 方法加载类
                // 自定义 ClassLoader 类需要重写 findClass 方法，否则会抛出 ClassNotFoundException 异常把加载任务下沉到子加载器去执行
                long t1 = System.nanoTime();
                c = findClass(name);

                // this is the defining class loader; record the stats
                PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                PerfCounter.getFindClasses().increment();
            }
        }
        if (resolve) {
            resolveClass(c);
        }
        return c;
    }
}

protected Class<?> findClass(String name) throws ClassNotFoundException {
    // 抛出 ClassNotFoundException 异常
    throw new ClassNotFoundException(name);
}
```

_双亲委派是翻译问题，不是指有两个加载器。_

## 逃逸分析

典型的对象逃逸：对象被复制给成员变量或者静态变量，可能被外部使用，此时变量就发生了逃逸。

```java
public class Foo {
    private Bar bar;
    
    public void init(){
        this.bar = new Bar();
    }
}
```

对象通过`return`语句返回，此时的程序并不能确定这个对象后续会不会被使用，外部的线程可以访问到这个变量，此时对象也发生了逃逸。

```java
public Object create() {
    Object obj = new Object();
    // do something
    return obj;
}
```

通过逃逸分析，HotSpot VM 能够分析变量作用域从而实现某些特殊的优化。

在 HotSpot VM 的源码中，可以看到逃逸分析系统是如何对对象的使用进行分类的：

```c
typedef enum {
  NoEscape = 1,    // An object does not escape method or thread and it is
                   // not passed to call. It could be replaced with scalar.

  ArgEscape = 2,   // An object does not escape method or thread but it is
                   // passed as argument to call or referenced by argument
                   // and it does not escape during call.
                   
  GlobalEscape = 3 // An object escapes the method or thread.
}
```

### 栈上分配

我们通过JVM内存分配可以知道JAVA中的对象都是在堆上进行分配，当对象没有被引用的时候，需要依靠GC进行回收内存，如果对象数量较多的时候，会给GC带来较大压力，也间接影响了应用的性能。
为了减少临时对象在堆内分配的数量，JVM通过逃逸分析确定该对象不会被外部访问。那就通过标量替换将该对象分解在栈上分配内存，这样该对象所占用的内存空间就可以随栈帧出栈而销毁，就减轻了垃圾回收的压力。

```java
public class Rect {
    private int w;
    private int h;

    public Rect(int w, int h) {
        this.w = w;
        this.h = h;
    }

    public int area() {
        return w * h;
    }

    public boolean isSameArea(Rect other) {
        return this.area() == other.area();
    }
}

public static void main(String[] args) {
    Random rand = new Random();
    int sameArea = 0;
    for (int i = 0; i < 100000000; i++) {
        Rect r1 = new Rect(rand.nextInt(5), rand.nextInt(5));
        Rect r2 = new Rect(rand.nextInt(5), rand.nextInt(5));
        if (r1.isSameArea(r2)) {
            sameArea++;
        }
    }
    System.out.println("Same area: " + sameArea);
}
```

上述代码创建了一亿对随机大小的矩形，并去计算有多少对是大小一样的。每次迭代都会创建一对新的矩形，你可能会认为一共创建2亿个`Rect`对象。

经过逃逸分析后发现这些对象不会发生逃逸，则这些不会在堆上分配空间，而是在栈上分配内存，对象所占用内存空间随栈帧出栈而释放，因此它们也不需要通过垃圾回收器来回收。

### 标量替换

- 标量：指不可分解的量，如：基本数据类型和`reference`类型
- 聚合量：指可继续分解的量，如：Java 对象

经过逃逸分析后发现一个对象不会被外部访问，并且该对象可以被分散，那么程序真正执行时将可能不会创建对象，而是在方法内部创建多个局部变量。

```java
public class Rect {
    private int w;
    private int h;

    public Rect(int w, int h) {
        this.w = w;
        this.h = h;
    }
}

/** 标量替换前 */
public static void main(String[] args) {
    Rect rect = new Rect(1, 1);
    System.out.println("width: " + rect.w + ", height: " + rect.h);
}

/** 标量替换后 */
public static void main2(String[] args) {
    int w = 1;
    int h = 1;
    System.out.println("width: " + w + ", height: " + h);
}
```

### 同步锁消除

经过逃逸分析发现一个对象被加了锁，但是只会被一个线程被访问到，会移除不可能存在共享资源竞争的锁，通过这种方式消除没有必要的锁，可以节省毫无意义的时间消耗。

被加锁的对象只有当前方法中使用：

```java
synchronized (new Object()) {
    run();
}
```
