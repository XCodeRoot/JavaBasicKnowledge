### 什么是 Fork/Join

Fork/Join 框架是一个实现了 ExecutorService 接口的多线程处理器，它专为那些可以通过递归分解成更细小的任务而设计，最大化的利用多核处理器来提高应用程序的性能。

与其他 ExecutorService 相关的实现相同的是，Fork/Join 框架会将任务分配给线程池中的线程。而与之不同的是，Fork/Join 框架在执行任务时使用了 **工作窃取算法**。

![](img/fork_join流程图.png)

### 工作窃取算法

工作窃取算法指的是在多线程执行不同任务队列的过程中，某个线程执行完自己队列的任务后从其他线程的任务队列里窃取任务来执行。

![](img/工作窃取算法运行流程图.png)

当一个线程窃取另一个线程的时候，为了减少两个任务线程之间的竞争，通常使用 **双端队列** 来存储任务。被窃取的任务线程都从双端队列的 **头部** 拿任务执行，而窃取其他任务的线程从双端队列的 **尾部** 执行任务。

另外，当一个线程在窃取任务时要是没有其他可用的任务了，这个线程会进入 **阻塞状态** 以等待再次“工作”。

### Fork/Join 的具体实现

在 Fork/Join 框架里提供了抽象类 ForkJoinTask 来实现任务。

#### ForkJoinTask

ForkJoinTask 是一个类似普通线程的实体，但是比普通线程轻量得多。

fork() 方法：使用线程池中的空闲线程异步提交任务，把任务推入当亲工作线程的工作队列里。

```java
public final ForkJoinTask<V> fork() {
    Thread t;
    // ForkJoinWorkerThread是执行ForkJoinTask的专有线程，由ForkJoinPool管理
    // 先判断当前线程是否是ForkJoin专有线程，如果是，则将任务push到当前线程所负责的队列里去
    if ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread)
        ((ForkJoinWorkerThread)t).workQueue.push(this);
    else
         // 如果不是则将线程加入队列
        // 没有显式创建ForkJoinPool的时候走这里，提交任务到默认的common线程池中
        ForkJoinPool.common.externalPush(this);
    return this;
}
```

join() 方法：等待处理任务的线程处理完毕，获得返回值。

```java
public final V join() {
    int s;
    // doJoin()方法来获取当前任务的执行状态
    if ((s = doJoin() & DONE_MASK) != NORMAL)
        // 任务异常，抛出异常
        reportException(s);
    // 任务正常完成，获取返回值
    return getRawResult();
}

/**
 * doJoin()方法用来返回当前任务的执行状态
 **/
private int doJoin() {
    int s; Thread t; ForkJoinWorkerThread wt; ForkJoinPool.WorkQueue w;
    // 先判断任务是否执行完毕，执行完毕直接返回结果（执行状态）
    return (s = status) < 0 ? s :
    // 如果没有执行完毕，先判断是否是ForkJoinWorkThread线程
    ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread) ?
        // 如果是，先判断任务是否处于工作队列顶端（意味着下一个就执行它）
        // tryUnpush()方法判断任务是否处于当前工作队列顶端，是返回true
        // doExec()方法执行任务
        (w = (wt = (ForkJoinWorkerThread)t).workQueue).
        // 如果是处于顶端并且任务执行完毕，返回结果
        tryUnpush(this) && (s = doExec()) < 0 ? s :
        // 如果不在顶端或者在顶端却没未执行完毕，那就调用awitJoin()执行任务
        // awaitJoin()：使用自旋使任务执行完成，返回结果
        wt.pool.awaitJoin(w, this, 0L) :
    // 如果不是ForkJoinWorkThread线程，执行externalAwaitDone()返回任务结果
    externalAwaitDone();
}
```

![](img/join流程图.png)

通常情况下，在创建任务的时候我们一般不直接继承 ForkJoinTask，而是继承它的子类 **RecursiveAction** 和 **RecursiveTask**。**RecursiveAction 可以看做是无返回值的 ForkJoinTask，RecursiveTask 是有返回值的 ForkJoinTask**。

#### ForkJoinPool

ForkJoinPool 是用于执行 ForkJoinTask 任务的执行（线程）池。

```java
@sun.misc.Contended
public class ForkJoinPool extends AbstractExecutorService {
    // 任务队列
    volatile WorkQueue[] workQueues;   

    // 线程的运行状态
    volatile int runState;  

    // 创建ForkJoinWorkerThread的默认工厂，可以通过构造函数重写
    public static final ForkJoinWorkerThreadFactory defaultForkJoinWorkerThreadFactory;

    // 公用的线程池，其运行状态不受shutdown()和shutdownNow()的影响
    static final ForkJoinPool common;

    // 私有构造方法，没有任何安全检查和参数校验，由makeCommonPool直接调用
    // 其他构造方法都是源自于此方法
    // parallelism: 并行度，
    // 默认调用java.lang.Runtime.availableProcessors() 方法返回可用处理器的数量
    private ForkJoinPool(int parallelism,
                         ForkJoinWorkerThreadFactory factory, // 工作线程工厂
                         UncaughtExceptionHandler handler, // 拒绝任务的handler
                         int mode, // 同步模式
                         String workerNamePrefix) { // 线程名prefix
        this.workerNamePrefix = workerNamePrefix;
        this.factory = factory;
        this.ueh = handler;
        this.config = (parallelism & SMASK) | mode;
        long np = (long)(-parallelism); // offset ctl counts
        this.ctl = ((np << AC_SHIFT) & AC_MASK) | ((np << TC_SHIFT) & TC_MASK);
    }

}
```

#### WorkQueue

双端队列，ForkJoinTask 存放在这里。

当工作线程在处理自己的工作队列时，会从队列首取任务来执行（FIFO）；如果是窃取其他队列的任务时，窃取的任务位于所属任务队列的队尾（LIFO）。

ForkJoinPool 与传统线程池最显著的区别就是它维护了一个**工作队列数组**（volatile WorkQueue[] workQueues，ForkJoinPool 中的**每个工作线程都维护着一个工作队列**）。

#### runState

ForkJoinPool 的运行状态。**SHUTDOWN** 状态用负数表示，其他用 2 的幂次表示。