# Thread 线程

## Thread 和 Runnable

`Thread`是类，`Runnable`是接口，实现`Runnable`接口的类可以不用是`Thread`的子类，通过实例化`Runnable`实例并将自身作为目标传入`Thread`即可运行。

大多数情况下，如果只想重写`run()`方法，而不重写`Thread`的方法，就应该使用`Runnable`接口，除非开发者想要修改或增强`Thread`的基本行为。

## Runnable 和 Callable

`Callable`接口是 Java 5 新增的接口，与`Runnable`接口非常相似，不同的是`Runnable`接口需要实现`void run()`方法，而`Callable`需要实现`V call()`方法，并且支持返回值。

```java
@FunctionalInterface
public interface Runnable {
    /**
     * When an object implementing interface <code>Runnable</code> is used
     * to create a thread, starting the thread causes the object's
     * <code>run</code> method to be called in that separately executing
     * thread.
     * <p>
     * The general contract of the method <code>run</code> is that it may
     * take any action whatsoever.
     *
     * @see     java.lang.Thread#run()
     */
    public abstract void run();
}
```

```java
@FunctionalInterface
public interface Callable<V> {
    /**
     * Computes a result, or throws an exception if unable to do so.
     *
     * @return computed result
     * @throws Exception if unable to compute a result
     */
    V call() throws Exception;
}
```

## 线程池

通常我们使用`Executors`工厂模式创建各种线程池，虽然返回的都是`ExecutorService`，但其背后实现逻辑大相径庭。

- Executors.newFixedThreadPool()
- Executors.newSingleThreadExecutor()
- Executors.newSingleThreadScheduledExecutor()
- Executors.newWorkStealingPool()

### ThreadPoolExecutor

`ThreadPoolExecutor`主要有以下几个属性：

- `corePoolSize`：核心线程数，线程池需要保持的线程数量，不管它们创建以后是不是空闲的，除非设置了`allowCoreThreadTimeOut`
- `maximumPoolSize`：最大线程数，线程池中最多允许创建线程数量
- `keepAliveTime`：存活时间，超过核心线程数的线程，超过存货时间后还没有接受到新的任务，则销毁线程
- `unit`：时间单位，`keepAliveTime`的时间单位
- `workQueue`：存放待执行任务的队列，当提交的任务数超过核心线程数大小后，再提交的任务就存放在这里。它仅用来存放提交的任务，所以这里就不要翻译为工作队列了好吗？不要给自己挖坑。

```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue) {
    this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
         Executors.defaultThreadFactory(), defaultHandler);
}
```

因为创建线程的开销比较大，所以只有满足以下所有条件时，才会创建额外的线程：

1. 正在运行的线程数量超过核心线程数
2. 阻塞队列已满
3. 正在运行的线程数量小于最大线程数

![process](images/java_thread_threadpoolexecutor_flow.png)

常用线程安全的队列：

![queue](images/java_thread_threadpoolexecutor_queues.png)

### ScheduledThreadPoolExecutor

`ScheduledThreadPoolExecutor`继承自`ThreadPoolExecutor`类，实现`ScheduledExecutorService`接口。
内部存在一个实现`BlockingQueue`接口的`DelayedWorkQueue`类，其原理基于小顶堆（与`DelayQueue`和`PriorityQueue`类似），每次从延迟队列中获取距离现在最近的一个任务。

### ForkJoinPool

`ForkJoinPool`是 Java 7 新加入的线程池，其基本原理是将一个大任务拆分成多个小任务，并行执行，再结合工作窃取模式提高整体的执行效率，充分利用CPU资源。

#### Fork/Join

`Fork/Join`并行方式是获取良好的并行计算性能的一种最简单同时也是最有效的设计技术。
`Fork/Join`并行算法是我们所熟悉的分治算法的并行版本，如同其他分治算法一样，总是会递归的、反复的划分子任务，直到这些子任务可以用足够简单的、短小的顺序方法来执行，典型的用法如下：

```c
Result solve(Problem problem) {
    if (problem is small) {
        directly solve problem
    } else {
        split problem into independent parts
        fork new subtasks to solve each part
        join all subtasks
        compose result from subresults
    }
}
```

`fork`操作将会启动一个新的并行子任务。

`join`操作会一直等待直到所有的子任务都结束。

#### work-stealing

`Fork/Join`框架的核心在于轻量级调度机制，采用了`work-stealing`的基本调度策略：

- 每个工作线程维护一个任务队列
- 队列以双端队列`Deque`的形式被维护，不仅支持后进先出`LIFO`（`push`、`pop`），还支持先进先出`FIFO`（`take`）
- 父任务`fork`操作产生的子任务会被追加到运行该父任务的工作线程对应的双端队列中
- 工作线程以后进先出`LIFO`的顺序处理队列中的任务（优先处理最新的任务）
- 当一个工作线程的本地队列没有任务时，会通过先进先出`FIFO`的规则尝试从其他的工作线程中窃取任务
- 当一个工作线程遇到`join`操作时，它会处理其他任务（如果有的话），直到目标任务已经完成，否则所有任务将在不阻塞的情况下运行至完成
- 当一个工作线程没有可执行的任务，且无法窃取任务时，它就会“退出”（通过`yield`、`sleep`或调整优先级），经过一段时间之后再度尝试，直到所有的工作线程都被处于空闲的状态

使用后进先出`LIFO`来处理每个工作线程自己的任务，而使用先进先出`FIFO`获取别的任务，这是一种被广泛使用的进行递归`Fork/Join`设计的一种调优手段。
让窃取任务的线程从队列拥有者相反的方向进行操作可以减少线程竞争，同样体现了递归分治算法的大任务优先策略。
因此，更早期被窃取的任务有可能会提供一个更大的单元任务，从而使得窃取线程能够在将来进行递归分解。

## TreadLocal

## wait 和 sleep

### wait和sleep的相同点和不同点

| wait                                                  | sleep      |
|:------------------------------------------------------|:-----------|
| 必须在`synchronized`中调用，同理`notify`和`notifyAll`方法         | 可以在任何地方调用  |
| 可以暂停线程                                                | 可以暂停线程     |
| 可以指定超时时间                                              | 可以指定休眠时间   |
| 未指定超时时间时不会自动苏醒，需要其他线程调用同一个对象上的`notify`或者`notifyAll`方法 | 根据超时时间自动苏醒 |
| 会释放锁                                                  | 不会释放锁      |
| 通常用于线程间交互和通信                                          | 通常用于暂停执行   |

#### 两个线程交替打印奇偶数

```java
private static volatile int count = 0;
private static final Object lock = new Object();

static class Turning implements Runnable {
    @Override
    public void run() {
        synchronized (lock) {
            while (count < 100) {
                System.out.println(Thread.currentThread().getName() + ":" + (++count));
                lock.notify();
                if (count < 100) {
                    try {
                        lock.wait();
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                }
            }
        }
    }
}

public static void main(String[] args) {
    new Thread(new Turning(), "奇数").start();
    new Thread(new Turning(), "偶数").start();
}
```