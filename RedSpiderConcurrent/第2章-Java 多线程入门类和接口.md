### Thread 类和 Runnable 接口

JDK 提供了 Thread 类和 Runnable 接口来让我们实现自己的“线程”类。

-   继承 Thread 类，并重写 run 方法；
-   实现 Runnable 接口的 run 方法。

#### 继承 Thread 类

```java
public class Demo {
    public static class MyThread extends Thread {
        @Override
        public void run() {
            System.out.println("MyThread");
        }
    }

    public static void main(String[] args) {
        Thread myThread = new MyThread();
        myThread.start();
    }
}
```

>   注意不可多次调用 `start()` 方法。在第一次调用 `start()` 方法后，再次调用 `start()` 方法会抛出 IllegalThreadStateException 异常。

#### 实现 Runnable 接口

```java
@FunctionalInterface
public interface Runnable {
    public abstract void run();
}
```

可以看到 Runnable 是一个函数式接口，这意味着可以使用 Java 8 的函数式编程来化简代码。

```java
public class Demo {
    public static class MyThread implements Runnable {
        @Override
        public void run() {
            System.out.println("MyThread");
        }
    }

    public static void main(String[] args) {

        new Thread(new MyThread()).start();

        // Java 8 函数式编程，可以省略MyThread类
        new Thread(() -> {
            System.out.println("Java 8 匿名内部类");
        }).start();
    }
}
```

#### Thread 类构造方法

```java
// Thread类源码 
// 片段1 - init方法
private void init(ThreadGroup g, Runnable target, String name,
                      long stackSize, AccessControlContext acc,
                      boolean inheritThreadLocals)

// 片段2 - 构造函数调用init方法
public Thread(Runnable target) {
    init(null, target, "Thread-" + nextThreadNum(), 0);
}

// 片段3 - 使用在init方法里初始化AccessControlContext类型的私有属性
this.inheritedAccessControlContext = 
    acc != null ? acc : AccessController.getContext();

// 片段4 - 两个对用于支持ThreadLocal的私有属性
ThreadLocal.ThreadLocalMap threadLocals = null;
ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
```

-   g：线程组，指定这个线程是在哪个线程组下；

-   target：指定要执行的任务；

-   name：线程的名字，多个线程的名字是可以重复的。如果不指定名字，见片段 2；

-   acc：见片段 3，用于初始化私有变量 inheritedAccessControlContext。

    >   它是一个私有变量，但是在 Thread 类里只有 init 方法对它进行初始化，在 exit 方法把它设为 null。

-   inheritThreadLocals：可继承的 ThreadLocal，见片段 4，Thread 类里面有两个私有属性来支持 ThreadLocal。

#### Thread 类的几个常用方法

-   `currentThread()`：静态方法，返回对当前正在执行的线程对象的引用；
-   `start()`：开始执行线程的方法，Java 虚拟机会调用线程内的 `run()` 方法；
-   `yield()`：yield 在英语里有放弃的意思，同样，这里的 `yield()` 指的是当前线程愿意让出对当前处理器的占用。 **这里需要注意的是，就算当前线程调用了 yield() 方法，程序在调度的时候，也还有可能继续运行这个线程的；**
-   `sleep()`：静态方法，使当前线程睡眠一段时间；
-   `join()`：使当前线程等待另一个线程执行完毕之后再继续执行，内部调用的是 Object 类的 wait 方法实现的。

#### Thread 类与 Runnable 接口的比较：

-   由于 Java “单继承，多实现”的特性，Runnable 接口使用起来比 Thread 更灵活。
-   Runnable 接口出现更符合面向对象，将线程单独进行对象的封装。
-   Runnable 接口出现，降低了线程对象和线程任务的耦合性。
-   如果使用线程时不需要使用 Thread 类的诸多方法，显然使用 Runnable 接口更为轻量。

### Callable、Future  与 FutureTask

#### Callable

Callable 与 Runnable 类似，同样是只有一个抽象方法的函数式接口。不同的是，Callable 提供的方法是**有返回值**的，而且支持**泛型**。

```java
@FunctionalInterface
public interface Callable<V> {
    V call() throws Exception;
}
```

Callable 一般是配合线程池工具 ExecutorService 来使用的。ExecutorService 可以使用 submit 方法来让一个 Callable 接口执行。它会返回一个 Future，后续的程序可以通过这个 Future 的 get 方法得到结果。

```java
// 自定义Callable
class Task implements Callable<Integer>{
    @Override
    public Integer call() throws Exception {
        // 模拟计算需要一秒
        Thread.sleep(1000);
        return 2;
    }
    
    public static void main(String args[]) throws Exception {
        // 使用
        ExecutorService executor = Executors.newCachedThreadPool();
        Task task = new Task();
        Future<Integer> result = executor.submit(task);
        // 注意调用get方法会阻塞当前线程，直到得到结果。
        // 所以实际编码中建议使用可以设置超时时间的重载get方法。
        System.out.println(result.get()); 
    }
}
```

#### Future 接口

```java
public abstract interface Future<V> {
    public abstract boolean cancel(boolean paramBoolean);
    public abstract boolean isCancelled();
    public abstract boolean isDone();
    public abstract V get() throws InterruptedException, ExecutionException;
    public abstract V get(long paramLong, TimeUnit paramTimeUnit)
            throws InterruptedException, ExecutionException, TimeoutException;
}
```

cancel 方法是试图取消一个线程的执行。注意是 **试图** 取消， **并不一定能取消成功**。参数 paramBoolean 表示是否采用中断的方式取消线程执行。

#### FutureTask 类

```java
public interface RunnableFuture<V> extends Runnable, Future<V> {
    /**
     * Sets this Future to the result of its computation
     * unless it has been cancelled.
     */
    void run();
}


// 自定义Callable，与上面一样
class Task implements Callable<Integer>{
    @Override
    public Integer call() throws Exception {
        // 模拟计算需要一秒
        Thread.sleep(1000);
        return 2;
    }
    public static void main(String args[]) throws Exception {
        // 使用
        ExecutorService executor = Executors.newCachedThreadPool();
        FutureTask<Integer> futureTask = new FutureTask<>(new Task());
        executor.submit(futureTask);
        System.out.println(futureTask.get());
    }
}
```

FutureTask 能够在高并发环境下 **确保任务只执行一次**。

#### FutureTask 的几个状态

```java
/**
  *
  * state可能的状态转变路径如下：
  * NEW -> COMPLETING -> NORMAL
  * NEW -> COMPLETING -> EXCEPTIONAL
  * NEW -> CANCELLED
  * NEW -> INTERRUPTING -> INTERRUPTED
  */
private volatile int state;
private static final int NEW          = 0;
private static final int COMPLETING   = 1;
private static final int NORMAL       = 2;
private static final int EXCEPTIONAL  = 3;
private static final int CANCELLED    = 4;
private static final int INTERRUPTING = 5;
private static final int INTERRUPTED  = 6;
```


