# JVM

## ClassLoader

JVM 中内置了三个重要的 ClassLoader，除了 BootstrapClassLoader 其他类加载器均由 Java 实现且全部继承自java.lang.ClassLoader：

### BootstrapClassLoader

引导类加载器，又称启动类加载器，是最顶层的类加载器，主要用来加载核心类，如：`rt.jar`、`resources.jar`、`charsets.jar`等。执行`java`的命令中使用`-Xbootclasspath`指定需要附加加载类的路径。

需要注意的是，它不是`java.lang.ClassLoader`的子类，而是由 JVM 自身实现的该类`C++`实现，Java 程序无法访问该加载器。

### ExtClassloader

扩展类加载器，主要负责加载扩展类库，默认加载`%JAVA_HOME%/jre/lib/ext/`目下的所有jar包和类，或者由`java.ext.dirs`系统属性指定的jar包，
放入这个目录下的jar包对所有`AppClassloader`都是可见的，因为`ExtClassloader`是`AppClassloader`的父加载器。

### AppClassloader

系统类加载器，又称应用程序类加载器，它负责在 JVM 启动时加载`classpath`，或者`java.class.path`系统属性，或者操作系统`CLASSPATH`属性所指定的jar包和类路径。

调用`ClassLoader.getSystemClassLoader()`可以获取该类加载器（`SystemClassLoader`即`AppClassloader`）。
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
ClassLoader.getSystemClassLoader().getParent()
```

一般我们都认为`ExtClassloader`的父类加载器是`BootstrapClassLoader`，但是其实他们之间根本是没有父子关系的，只是在`ExtClassloader`找不到要加载类时候会去委托`BootstrapClassLoader`去加载。

_注意，这里说的父子关系并不是继承关系，而是通过定义`private final ClassLoader parent`属性关联。_

## 双亲委派

Java 类加载器使用的是双亲委托机制，也就是子类加载器在加载一个类时候会让父类来加载，那么问题来了，为什么使用这种方式呢？因为这样可以避免重复加载，当父类加载器已经加载了该类的时候，就没有必要用子类加载器再加载一次。
考虑到安全因素，我们试想一下，如果不使用这种委托模式，那我们就可以随时使用自定义的`String`来动态替代核心`API`中定义的类型，这样会存在非常大的安全隐患，而双亲委托的机制就可以避免这种情况，因为`String`已经在启动时就被`BootstrapClassLoader`加载，
所以用户自定义的`ClassLoader`永远也无法加载一个自己写的`String`，除非`JDK`源码中`ClassLoader`搜索类的默认算法。

```java
protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
    synchronized (getClassLoadingLock(name)) {
        // First, check if the class has already been loaded
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            long t0 = System.nanoTime();
            try {
                if (parent != null) {
                    c = parent.loadClass(name, false);
                } else {
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
                // ClassNotFoundException thrown if class not found
                // from the non-null parent class loader
            }

            if (c == null) {
                // If still not found, then invoke findClass in order
                // to find the class.
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
```

## 逃逸分析

## GC

### CMS

### G1

### ZGC