# Singleton 单例

## 双重检查锁

单例变量必须使用`volatile`修饰，保证可见性，防止指令重排序，避免其他线程拿到没完成初始化的对象。

```java
public class Singleton {
    private static volatile Singleton instance;
    
    private Singleton() {
    }
    
    public static Singleton getInstance() {
        if (instance == null) {
            if (instance == null) {
                synchronized (Singleton.class) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

## 静态内部类

在 Java 中推荐使用静态内部类实现单例：

```java
public class Singleton {
    private static class Inner {
        private static final Singleton instance = new Singleton();
    }

    public static Singleton getInstance() {
        return Inner.instance;
    }
}
```

当一个类被加载时，其内部类不会被同时加载，当且仅当其某个静态成员（静态域、构造器、静态方法等）被调用时发生。
也就是说只有在调用`Singleton.getInstance()`的时候`Singleton.Inner.class`才会被加载。

虚拟机会保证一个类的初始化方法在多线程环境中被正确的加锁同步，如果多个线程同时去初始化同一个类，那么只会有一个线程去执行这个类的初始化方法，其他线程都需要阻塞等待，直到活动线程执行初始化方法完毕。
如果在一个类的初始化方法中有耗时很长的操作，就有可能造成多个线程阻塞（在实际应用中这种阻塞往往是很隐蔽的），这就是他线程安全的原因。

需要注意的是，其他线程虽然会被阻塞，但如果执行初始化方法的那条线程退出初始化方法后，其他线程唤醒之后不会再次进入初始化方法，因为同一个类加载器下，一个类只会被初始化一次。
