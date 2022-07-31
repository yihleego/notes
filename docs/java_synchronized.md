# synchronized

Java 中有两种加锁的方式：一种是用`synchronized`关键字，另一种是用`Lock`接口的实现类。

|          | synchronized | Lock                                                            |
|:---------|:-------------|:----------------------------------------------------------------|
| 作用域      | 方法和代码块       | 代码块                                                             |
| 实现方式     | 对象头          | AbstractQueuedSynchronizer                                      |
| 是否可重入    | 可重入          | 根据实现类，例如`ReentrantLock`和`ReentrantReadWriteLock`可重入             |
| 是否是公平锁   | 非公平锁         | 根据实现类，例如`ReentrantLock`和`ReentrantReadWriteLock`可通过构造方法指定公平或非公平 |
| 是否可主动释放锁 | 不可主动释放       | 可主动释放，调用`unlock()`方法                                            |

## synchronized 使用方式

### 作用对象

作用在代码块并指定对象时，锁指定对象。

```java
public synchronized void foo() {
    synchronized (bar) {
        // ...
    }
}
```

### 作用方法

修饰在非静态方法上时，锁当前对象。

```java
public synchronized void foo() {
    // ...
}
```

### 作用静态方法

修饰在静态方法上时，锁当前类。

```java
public static synchronized void foo() {
    // ...
}
```

### 作用类

作用在代码块并指定类时，锁指定类。

```java
public synchronized void foo() {
    synchronized (Bar.class) {
        // ...
    }
}
```

## synchronized 使用方式的区别

通过`javap -v SynchronizedStyle.java`命令反编译以下类：

```
public class SynchronizedStyle {
    private final Object lock = new Object();

    public void syncObject() {
        synchronized (lock) {
            System.out.println("syncObject");
        }
    }

    public synchronized void syncMethod() {
        System.out.println("syncMethod");
    }


    public static synchronized void syncStaticMethod() {
        System.out.println("syncObject");
    }

    public void syncClass() {
        synchronized (SynchronizedStyle.class) {
            System.out.println("syncBlock");
        }
    }
}
```

```text
Classfile /synchronized-markword/target/classes/io/leego/test/SynchronizedStyle.class
  Last modified 2022年7月29日; size 1007 bytes
  SHA-256 checksum e7727ab627c0fb18699ce95d4405689f7ceaebf84bb71ecd24bc940b1bc794c3
  Compiled from "SynchronizedStyle.java"
public class io.leego.test.SynchronizedStyle
  minor version: 0
  major version: 52
  flags: (0x0021) ACC_PUBLIC, ACC_SUPER
  this_class: #8                          // io/leego/test/SynchronizedStyle
  super_class: #2                         // java/lang/Object
  interfaces: 0, fields: 1, methods: 5, attributes: 1
Constant pool:
   #1 = Methodref          #2.#3          // java/lang/Object."<init>":()V
   #2 = Class              #4             // java/lang/Object
   #3 = NameAndType        #5:#6          // "<init>":()V
   #4 = Utf8               java/lang/Object
   #5 = Utf8               <init>
   #6 = Utf8               ()V
   #7 = Fieldref           #8.#9          // io/leego/test/SynchronizedStyle.lock:Ljava/lang/Object;
   #8 = Class              #10            // io/leego/test/SynchronizedStyle
   #9 = NameAndType        #11:#12        // lock:Ljava/lang/Object;
  #10 = Utf8               io/leego/test/SynchronizedStyle
  #11 = Utf8               lock
  #12 = Utf8               Ljava/lang/Object;
  #13 = Fieldref           #14.#15        // java/lang/System.out:Ljava/io/PrintStream;
  #14 = Class              #16            // java/lang/System
  #15 = NameAndType        #17:#18        // out:Ljava/io/PrintStream;
  #16 = Utf8               java/lang/System
  #17 = Utf8               out
  #18 = Utf8               Ljava/io/PrintStream;
  #19 = String             #20            // syncObject
  #20 = Utf8               syncObject
  #21 = Methodref          #22.#23        // java/io/PrintStream.println:(Ljava/lang/String;)V
  #22 = Class              #24            // java/io/PrintStream
  #23 = NameAndType        #25:#26        // println:(Ljava/lang/String;)V
  #24 = Utf8               java/io/PrintStream
  #25 = Utf8               println
  #26 = Utf8               (Ljava/lang/String;)V
  #27 = String             #28            // syncMethod
  #28 = Utf8               syncMethod
  #29 = String             #30            // syncBlock
  #30 = Utf8               syncBlock
  #31 = Utf8               Code
  #32 = Utf8               LineNumberTable
  #33 = Utf8               LocalVariableTable
  #34 = Utf8               this
  #35 = Utf8               Lio/leego/test/SynchronizedStyle;
  #36 = Utf8               StackMapTable
  #37 = Class              #38            // java/lang/Throwable
  #38 = Utf8               java/lang/Throwable
  #39 = Utf8               syncStaticMethod
  #40 = Utf8               syncClass
  #41 = Utf8               SourceFile
  #42 = Utf8               SynchronizedStyle.java
{
  public io.leego.test.SynchronizedStyle();
    descriptor: ()V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=3, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: aload_0
         5: new           #2                  // class java/lang/Object
         8: dup
         9: invokespecial #1                  // Method java/lang/Object."<init>":()V
        12: putfield      #7                  // Field lock:Ljava/lang/Object;
        15: return
      LineNumberTable:
        line 6: 0
        line 7: 4
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      16     0  this   Lio/leego/test/SynchronizedStyle;

  public void syncObject();
    descriptor: ()V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=2, locals=3, args_size=1
         0: aload_0
         1: getfield      #7                  // Field lock:Ljava/lang/Object;
         4: dup
         5: astore_1
         6: monitorenter
         7: getstatic     #13                 // Field java/lang/System.out:Ljava/io/PrintStream;
        10: ldc           #19                 // String syncObject
        12: invokevirtual #21                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        15: aload_1
        16: monitorexit
        17: goto          25
        20: astore_2
        21: aload_1
        22: monitorexit
        23: aload_2
        24: athrow
        25: return
      Exception table:
         from    to  target type
             7    17    20   any
            20    23    20   any
      LineNumberTable:
        line 10: 0
        line 11: 7
        line 12: 15
        line 13: 25
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      26     0  this   Lio/leego/test/SynchronizedStyle;
      StackMapTable: number_of_entries = 2
        frame_type = 255 /* full_frame */
          offset_delta = 20
          locals = [ class io/leego/test/SynchronizedStyle, class java/lang/Object ]
          stack = [ class java/lang/Throwable ]
        frame_type = 250 /* chop */
          offset_delta = 4

  public synchronized void syncMethod();
    descriptor: ()V
    flags: (0x0021) ACC_PUBLIC, ACC_SYNCHRONIZED
    Code:
      stack=2, locals=1, args_size=1
         0: getstatic     #13                 // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #27                 // String syncMethod
         5: invokevirtual #21                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: return
      LineNumberTable:
        line 16: 0
        line 17: 8
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       9     0  this   Lio/leego/test/SynchronizedStyle;

  public static synchronized void syncStaticMethod();
    descriptor: ()V
    flags: (0x0029) ACC_PUBLIC, ACC_STATIC, ACC_SYNCHRONIZED
    Code:
      stack=2, locals=0, args_size=0
         0: getstatic     #13                 // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #19                 // String syncObject
         5: invokevirtual #21                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: return
      LineNumberTable:
        line 21: 0
        line 22: 8

  public void syncClass();
    descriptor: ()V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=2, locals=3, args_size=1
         0: ldc           #8                  // class io/leego/test/SynchronizedStyle
         2: dup
         3: astore_1
         4: monitorenter
         5: getstatic     #13                 // Field java/lang/System.out:Ljava/io/PrintStream;
         8: ldc           #29                 // String syncBlock
        10: invokevirtual #21                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        13: aload_1
        14: monitorexit
        15: goto          23
        18: astore_2
        19: aload_1
        20: monitorexit
        21: aload_2
        22: athrow
        23: return
      Exception table:
         from    to  target type
             5    15    18   any
            18    21    18   any
      LineNumberTable:
        line 25: 0
        line 26: 5
        line 27: 13
        line 28: 23
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      24     0  this   Lio/leego/test/SynchronizedStyle;
      StackMapTable: number_of_entries = 2
        frame_type = 255 /* full_frame */
          offset_delta = 18
          locals = [ class io/leego/test/SynchronizedStyle, class java/lang/Object ]
          stack = [ class java/lang/Throwable ]
        frame_type = 250 /* chop */
          offset_delta = 4
}
SourceFile: "SynchronizedStyle.java"
```

从反编译结果可以得：

- 同步代码块：`javac`编译时，会生成`monitorenter`和`monitorexit`指令，分别对应进入同步代码块和退出同步代码块。存在两个`monitorexit`指令的原因是，保证抛出异常的情况下也可以释放锁，为同步代码块包装了隐式的`try-finally`，在`finally`中调用了`monitorexit`指令。
- 同步方法：`javac`编译时，会通过`ACC_SYNCHRONIZED`关键字修饰，JVM 执行方法时，发现存在`ACC_SYNCHRONIZED`关键字，则会先尝试获取锁。

在 JVM 底层中，这两种`synchronized`方式的实现基本相同。

## synchronized 锁类型

在 Java 6 版本之前，Java 仅支持[重量级锁](#重量级锁-Heavyweight-Locking)，在 Java 6 版本中引入了[偏向锁](#偏向锁-Biased-Locking)和[轻量级锁](#轻量级锁-Lightweight-Locking)。

引入的目的是为了在无锁竞争或少竞争的情况下，避免使用重量级锁。因为重量级锁依赖于系统级别的同步函数，在 Linux 中使用`mutex`互斥锁，底层实现依赖于`futex`，这些同步函数都涉及到用户态和内核态的切换、进程的上下文切换，会带来一定的性能开销。

## Mark Word

在 Java 中任意对象都可以当作锁，因此需要维护一个对象和锁的映射关系，比如，当前哪个线程持有锁，哪些线程在等待。
Java 选择将这个映射关系存储在对象头中，其中还包括了 HashCode、GC 相关的信息，该区域被称为 Mark Word。

对象头中除了 Mark Word，还有储存了指向该对象所属类对象的指针。对于数组，还会储存记录数组长度的信息。

为了能在有限的空间里存储下更多的数据，其存储格式不固定。

32 位虚拟机中对象头的 Mark Word 结构：

![markword_32](images/java_jvm_markword_32.png)

64 位虚拟机中对象头的 Mark Word 结构：

![markword_64](images/java_jvm_markword_64.png)

- 偏向锁：Mark Word 储存了偏向的线程ID，偏向锁标识为`1`，锁标识为`01`。
- 轻量级锁：Mark Word 储存了指向线程栈帧中 Lock Record 的指针，偏向锁标识为`0`，锁标识为`00`。
- 重量级锁：Mark Word 储存了指向堆中的 Monitor 对象的指针，偏向锁标识为`0`，锁标识为`10`。

不同 Java 版本中，使用`synchronized`对应 Mark Word 的变化可以看[这篇文档](https://github.com/yihleego/synchronized-markword)。

## 管程 Monitor

Monitor 被翻译为监视器或管程，每个 Java 对象都可以关联一个 Monitor 对象，使用 synchronized 获取对象的重量级锁之后，该对象头的 Mark Word 中就被设置指向 Monitor 对象的指针。Monitor 的结构如下：

![monitor](images/java_jvm_monitor.png)

1. 默认情况下，Monitor 的 Owner 为 null
2. 当 Thread-2 进入同步代码块时，就会将 Monitor 的 Owner 设置为 Thread-2
3. 在 Thread-2 同步的过程中，如果 Thread-3、Thread-4、Thread-5 也进入同步代码块时，就会被放入 EntryList 中，进入阻塞状态
4. 在 Thread-2 退出同步代码块时，然后唤醒 EntryList 中等待的线程来竞争锁，竞争的时是非公平的
5. 图中 WaitSet 中的存放的 Thread-0 和 Thread-1 是在同步代码块中，调用了`Object.wait()`的线程
    - `Object.wait()`会使线程等待，WaitSet 存放等待的线程
    - `Object.notify()`会唤醒一个 WaitSet 中的线程
    - `Object.notifyAll()`会唤醒 WaitSet 中的所有线程

这些方法用于线程之间进行协作，属于`Object`对象的方法，因此调用这些方法必须在同步代码块内。

## 安全点 SafePoint

SafePoint 在 HotSpot VM 中是一个核心的技术点，所谓安全点指的是代码执行过程中被选择出来的一些位置，当需要执行一些要 STW（Stop The World） 的操作的时候，这些位置⽤于线程进⼊这些位置并等待系统执⾏完成 STW 操作。
所以，安全点不能太少也不能太多，安全点过少会导致那些需要执⾏ STW 操作的程序需要等待太久，安全点太多⼜会导致程序执⾏时需要频繁检查安全点，导致系统负载升⾼。

在 HotSpot VM 中，需要 STW 的操作典型的是 GC，GC 时需要所有线程同时进入安全点，并阻塞等待 GC 处理完，然后再让所有线程继续执行。

除了 GC 还有一些场景会让所有线程进入 SafePoint，即发生 STW：

1. 定时进入 SafePoint
2. 使用 jstack、jmap、jstat 等命令
3. 偏向锁撤销（不一定会引发整体的 STW）
4. Java Instrument 导致的 Agent 加载以及类的重定义
5. 当发生 JIT 编译优化或者去优化，需要 OSR 或者 Bailout 或者清理代码缓存的时候
6. 如果开启了 JFR 的 OldObject 采集，这个是定时采集一些存活时间比较久的对象

## 偏向锁 Biased Locking

偏向锁能够减少无竞争锁定时的开销，其目的是假定该锁一直由某个特定线程持有，直到另一个线程尝试获取它，这样就可以避免同一对象的后续同步操作执行 CAS 操作，减少了获取锁和释放锁的次数。

Java 15 版本弃用了偏向锁：[JEP 374: Disable and Deprecate Biased Locking](https://openjdk.java.net/jeps/374)

> 从历史上看，偏向锁使得 JVM 的性能得到了显著改善。但是过去看到的性能提升在今天远不那么明显。  
> 许多受益于偏向锁的应用程序是使用早期 Java 集合 API 的旧的遗留应用程序，这些 API 在每次访问时都会同步（例如：Hashtable 和 Vector）。  
> 较新的应用程序通常使用 Java 1.2 中针对单线程场景引入的非同步集合（例如：HashMap 和 ArrayList），或者使用 Java 5 中针对多线程场景引入的性能更高的并发数据结构。  
> 这意味着如果更新代码以使用这些较新的类，由于不必要的同步而受益于偏向锁的应用程序可能会看到性能改进。此外，围绕线程池队列和工作线程构建的应用程序通常在禁用偏向锁的情况下性能更好。
>
> 偏向锁在同步子系统中引入了许多复杂的代码，并且还侵入了其他 HotSpot 组件。这种复杂性是理解代码各个部分的障碍，也是在同步子系统内进行重大设计更改的障碍。为此，我们希望禁用、弃用并最终移除对偏向锁的支持。

_在 Java 15 之前，偏向锁始终处于启用状态且可用，可以通过`-XX:-UseBiasedLocking`禁用偏向锁。在 Java 15 及之后，启动 HotSpot 时将不再启用偏向锁，除非在命令行中设置`-XX:+UseBiasedLocking`启用偏向锁。_

_通常，在程序启动后，偏向锁默认不会立即生效，而是存在几秒延迟，可以通过命令`-XX:BiasedLockingStartupDelay=0`关闭延迟。_

### 获取偏向锁流程

示例中源码基于 OpenJDK 8u312-GA [/src/share/vm/interpreter/bytecodeInterpreter.cpp#L1816](https://hg.openjdk.java.net/jdk8u/jdk8u/hotspot/file/8aac6d08b58e/src/share/vm/interpreter/bytecodeInterpreter.cpp#l1816)

```cpp
CASE(_monitorenter): {
  // lockee 即为锁对象
  oop lockee = STACK_OBJECT(-1);
  // derefing's lockee ought to provoke implicit null check
  CHECK_NULL(lockee);
  // find a free monitor or one already allocated for this object
  // if we find a matching object then we need a new monitor
  // since this is recursive enter
  BasicObjectLock* limit = istate->monitor_base();
  BasicObjectLock* most_recent = (BasicObjectLock*) istate->stack_base();
  BasicObjectLock* entry = NULL;
  while (most_recent != limit ) {
    if (most_recent->obj() == NULL) entry = most_recent;
    else if (most_recent->obj() == lockee) break;
    most_recent++;
  }
  if (entry != NULL) {
    entry->set_obj(lockee);
    int success = false;
    uintptr_t epoch_mask_in_place = (uintptr_t)markOopDesc::epoch_mask_in_place;

    markOop mark = lockee->mark();
    intptr_t hash = (intptr_t) markOopDesc::no_hash;
    // implies UseBiasedLocking
    if (mark->has_bias_pattern()) {
      uintptr_t thread_ident;
      uintptr_t anticipated_bias_locking_value;
      thread_ident = (uintptr_t)istate->thread();
      anticipated_bias_locking_value =
        (((uintptr_t)lockee->klass()->prototype_header() | thread_ident) ^ (uintptr_t)mark) &
        ~((uintptr_t) markOopDesc::age_mask_in_place);

      if  (anticipated_bias_locking_value == 0) {
        // already biased towards this thread, nothing to do
        if (PrintBiasedLockingStatistics) {
          (* BiasedLocking::biased_lock_entry_count_addr())++;
        }
        success = true;
      }
      else if ((anticipated_bias_locking_value & markOopDesc::biased_lock_mask_in_place) != 0) {
        // try revoke bias
        markOop header = lockee->klass()->prototype_header();
        if (hash != markOopDesc::no_hash) {
          header = header->copy_set_hash(hash);
        }
        if (Atomic::cmpxchg_ptr(header, lockee->mark_addr(), mark) == mark) {
          if (PrintBiasedLockingStatistics)
            (*BiasedLocking::revoked_lock_entry_count_addr())++;
        }
      }
      else if ((anticipated_bias_locking_value & epoch_mask_in_place) !=0) {
        // try rebias
        markOop new_header = (markOop) ( (intptr_t) lockee->klass()->prototype_header() | thread_ident);
        if (hash != markOopDesc::no_hash) {
          new_header = new_header->copy_set_hash(hash);
        }
        if (Atomic::cmpxchg_ptr((void*)new_header, lockee->mark_addr(), mark) == mark) {
          if (PrintBiasedLockingStatistics)
            (* BiasedLocking::rebiased_lock_entry_count_addr())++;
        }
        else {
          CALL_VM(InterpreterRuntime::monitorenter(THREAD, entry), handle_exception);
        }
        success = true;
      }
      else {
        // try to bias towards thread in case object is anonymously biased
        markOop header = (markOop) ((uintptr_t) mark & ((uintptr_t)markOopDesc::biased_lock_mask_in_place |
                                                        (uintptr_t)markOopDesc::age_mask_in_place |
                                                        epoch_mask_in_place));
        if (hash != markOopDesc::no_hash) {
          header = header->copy_set_hash(hash);
        }
        markOop new_header = (markOop) ((uintptr_t) header | thread_ident);
        // debugging hint
        DEBUG_ONLY(entry->lock()->set_displaced_header((markOop) (uintptr_t) 0xdeaddead);)
        if (Atomic::cmpxchg_ptr((void*)new_header, lockee->mark_addr(), header) == header) {
          if (PrintBiasedLockingStatistics)
            (* BiasedLocking::anonymously_biased_lock_entry_count_addr())++;
        }
        else {
          CALL_VM(InterpreterRuntime::monitorenter(THREAD, entry), handle_exception);
        }
        success = true;
      }
    }

    // traditional lightweight locking
    if (!success) {
      markOop displaced = lockee->mark()->set_unlocked();
      entry->lock()->set_displaced_header(displaced);
      bool call_vm = UseHeavyMonitors;
      if (call_vm || Atomic::cmpxchg_ptr(entry, lockee->mark_addr(), displaced) != displaced) {
        // Is it simple recursive case?
        if (!call_vm && THREAD->is_lock_owned((address) displaced->clear_lock_bits())) {
          entry->lock()->set_displaced_header(NULL);
        } else {
          CALL_VM(InterpreterRuntime::monitorenter(THREAD, entry), handle_exception);
        }
      }
    }
    UPDATE_PC_AND_TOS_AND_CONTINUE(1, -1);
  } else {
    istate->set_msg(more_monitors);
    UPDATE_PC_AND_RETURN(0); // Re-execute
  }
}
```

### 释放偏向锁流程

TODO

### 撤销偏向锁流程

TODO

### 批量重偏向与批量撤销

TODO

## 轻量级锁 Lightweight Locking

轻量级锁对少量线程竞争同一个资源并且他们的操作时间比较短的场景性能较好，没有竞争到锁的线程会自旋固定的次数来获取轻量级锁，不会阻塞线程。因为阻塞线程需要 CPU 从用户态转到内核态，其代价较大。

轻量级锁的获取过程：

1. 判断对象头的 Mark Word 锁标识是否为`01`无锁状态，如果当前没有锁，虚拟机首先在当前线程的栈帧中分配一个 Lock Record，用于存储锁对象的 Mark Word 的备份，官方称之为 Displaced Mark Word。
2. 复制对象头的 Mark Word 到 Lock Record 中。
3. 尝试`CAS`操作将 Mark Word 更新为指向 Lock Record 的指针，根据结果判断是否需要升级锁：
    - 如果更新操作成功，表示线程可以获取轻量级锁，将 Mark Word 锁标识设置为`00`。
    - 如果更新操作失败，表示锁已经被其他线程持有，当前线程通过自旋尝试获取锁，自旋一定次数后，将锁膨胀为重量级锁，将 Mark Word 锁标识设置为`10`，然后线程进入阻塞。
4. 如果对象头的 Mark Word 锁标识已经是`00`轻量级锁，再判断对象头的 Mark Word 是否指向当前线程的栈帧的 Lock Record，如果是，则表示当前线程已经持有了该锁，即锁的可重入。

轻量级锁的释放过程：

1. 通过`CAS`操作尝试将当前线程的 Displaced Mark Word 替换对象头的 Mark Word。
    - 如果替换成功，表示整个同步过程已经完成，且释放了锁。
    - 如果替换失败，表示存在其他线程尝试过获取该锁，此时锁为重量级锁，对象头的 Mark Word 指向重量级锁 Monitor 的指针，需要唤醒被挂起的线程。

![java_synchronized_lightweight_locking](images/java_synchronized_lightweight_locking.png)

## 重量级锁 Heavyweight Locking

当一个轻量级锁自旋超过一定次数（默认 10 次），或被三个及以上的线程竞争的时候，轻量级锁就会膨胀为重量级锁。

![java_synchronized_heavyweight_locking](images/java_synchronized_heavyweight_locking.png)

## synchronized 锁粗化

将多个连续的加锁、解锁操作连接在一起，扩展成一个范围更大的锁，避免频繁的加锁解锁操作。

例如在循环中使用`synchronized`：

```java
for (int i = 0; i < 1000; i++) {
    synchronized (this) {
        i++;
    }
}
```

优化后将锁的粒度粗化（近似代码）：

```java
synchronized (this) {
    for (int i = 0; i < 1000; i++) {
        i++;
    }
}
```

## synchronized 锁消除

虚拟机在 JIT 编译时，通过对运行上下文的扫描，经过逃逸分析发现一个对象被加了锁，但是只会被一个线程被访问到，会移除不可能存在共享资源竞争的锁，
通过这种方式消除没有必要的锁，可以节省毫无意义的时间消耗。

被加锁的对象只有当前方法中使用：

```java
synchronized (new Object()) {
    run();
}
```

## 悲观锁与乐观锁

悲观锁与乐观锁并不是特指某个锁，而是在并发情况下的两种不同策略。

- 悲观锁 (Pessimistic Lock): 读取或修改数据时会获取锁，若锁被别的线程持就等待或返回失败。
- 乐观锁 (Optimistic Lock): 乐观锁不是锁，而是在修改数据时，判断数据是否被修改过，若被修改过就进行重试或返回失败。

悲观锁阻塞事务，乐观锁回滚重试，它们适用于不同的场景。
乐观锁适用于写比较少的情况下，即冲突真的很少发生的时候，这样可以省去锁的开销，加大了系统的整个吞吐量。
若经常产生冲突，不断的进行重试造成性能浪费，这种情况下用悲观锁就比较合适。
