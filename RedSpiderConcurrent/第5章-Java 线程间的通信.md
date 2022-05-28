### 锁与同步

在 Java 中，锁的概念都是基于对象的，所以又经常称它为对象锁。线程同步是线程之间按照 **一定的顺序** 执行。

### 等待、通知机制

Java 多线程的等待/通知机制是基于 Object 类的 `wait()` 方法和 `notify()`，`notifyAll()` 方法来实现的。需要注意的是等待、通知机制使用的是使用同一个对象锁，如果两个线程使用的是不同的对象锁，那它们之间是不能用等待/通知机制通信的。

### 信号量

volatile 关键字能够保证内存的可见性，如果用 volatile 关键字声明了一个变量，在一个线程里面改变了这个变量的值，那其它线程是立马可见更改后的值的。

### 管道

管道是基于“管道流”的通信方式。JDK 提供了 PipedWriter、 PipedReader、 PipedOutputStream、 PipedInputStream。其中，前面两个是基于字符的，后面两个是基于字节流的。

```java
public class Pipe {
    static class ReaderThread implements Runnable {
        private PipedReader reader;

        public ReaderThread(PipedReader reader) {
            this.reader = reader;
        }

        @Override
        public void run() {
            System.out.println("this is reader");
            int receive = 0;
            try {
                while ((receive = reader.read()) != -1) {
                    System.out.print((char)receive);
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    static class WriterThread implements Runnable {

        private PipedWriter writer;

        public WriterThread(PipedWriter writer) {
            this.writer = writer;
        }

        @Override
        public void run() {
            System.out.println("this is writer");
            int receive = 0;
            try {
                writer.write("test");
            } catch (IOException e) {
                e.printStackTrace();
            } finally {
                try {
                    writer.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    public static void main(String[] args) throws IOException, InterruptedException {
        PipedWriter writer = new PipedWriter();
        PipedReader reader = new PipedReader();
        writer.connect(reader); // 这里注意一定要连接，才能通信

        new Thread(new ReaderThread(reader)).start();
        Thread.sleep(1000);
        new Thread(new WriterThread(writer)).start();
    }
}

// 输出：
this is reader
this is writer
test
```

可以简单分析一下这个示例代码的执行流程：

1.  线程 ReaderThread 开始执行，
2.  线程 ReaderThread 使用管道 `reader.read()` 进入”阻塞“，
3.  线程 WriterThread 开始执行，
4.  线程 WriterThread 用 `writer.write("test")` 往管道写入字符串，
5.  线程 WriterThread 使用 `writer.close()` 结束管道写入，并执行完毕，
6.  线程 ReaderThread 接受到管道输出的字符串并打印，
7.  线程 ReaderThread 执行完毕。

### 其他通信相关

#### join 方法

`join()` 方法是 Thread 类的一个实例方法。它的作用是让当前线程陷入“等待”状态，等 join 的这个线程执行完成后，再继续执行当前线程。

```java
public class Join {
    static class ThreadA implements Runnable {

        @Override
        public void run() {
            try {
                System.out.println("我是子线程，我先睡一秒");
                Thread.sleep(1000);
                System.out.println("我是子线程，我睡完了一秒");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        Thread thread = new Thread(new ThreadA());
        thread.start();
        thread.join();
        System.out.println("如果不加join方法，我会先被打出来，加了就不一样了");
    }
}
```

>   注意 `join()` 方法有两个重载方法，一个是 `join(long)`， 一个是 `join(long, int)`。
>
>   实际上，通过源码会发现，`join()` 方法及其重载方法底层都是利用了 `wait(long)` 这个方法。
>
>   对于 `join(long, int)`，通过查看源码（JDK 1.8）发现，底层并没有精确到纳秒，而是对第二个参数做了简单的判断和处理。

### sleep 方法

sleep 方法是 Thread 类的一个静态方法。它的作用是让当前线程睡眠一段时间。

**sleep 方法是不会释放当前的锁的，而 wait 方法会。**

它们还有这些区别：

-   wait 可以指定时间，也可以不指定；而 sleep 必须指定时间；
-   wait 释放 cpu 资源，同时释放锁；sleep 释放 cpu 资源，但是不释放锁，所以易死锁；
-   wait 必须放在同步块或同步方法中，而 sleep 可以在任意位置。

#### ThreadLocal 类

ThreadLocal 是一个本地线程副本变量工具类。内部是一个 **弱引用** 的 Map 来维护。严格来说，ThreadLocal 类并不属于多线程间的通信，而是让每个线程有自己”独立“的变量，线程之间互不影响。它为每个线程都创建一个 **副本**，每个线程可以访问自己内部的副本变量。

```java
public class ThreadLocalDemo {
    static class ThreadA implements Runnable {
        private ThreadLocal<String> threadLocal;

        public ThreadA(ThreadLocal<String> threadLocal) {
            this.threadLocal = threadLocal;
        }

        @Override
        public void run() {
            threadLocal.set("A");
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("ThreadA输出：" + threadLocal.get());
        }

        static class ThreadB implements Runnable {
            private ThreadLocal<String> threadLocal;

            public ThreadB(ThreadLocal<String> threadLocal) {
                this.threadLocal = threadLocal;
            }

            @Override
            public void run() {
                threadLocal.set("B");
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("ThreadB输出：" + threadLocal.get());
            }
        }

        public static void main(String[] args) {
            ThreadLocal<String> threadLocal = new ThreadLocal<>();
            new Thread(new ThreadA(threadLocal)).start();
            new Thread(new ThreadB(threadLocal)).start();
        }
    }
}

// 输出：
ThreadA输出：A
ThreadB输出：B
```

如果开发者希望将类的某个静态变量（user ID 或者 transaction ID）与线程状态关联，则可以考虑使用 ThreadLocal。

最常见的 ThreadLocal 使用场景为用来解决数据库连接、Session 管理等。数据库连接和 Session 管理涉及多个复杂对象的初始化和关闭。如果在每个线程中声明一些私有变量来进行操作，那这个线程就变得不那么“轻量”了，需要频繁的创建和关闭连接。

#### InheritableThreadLocal

InheritableThreadLocal 类与 ThreadLocal 类稍有不同，Inheritable 是继承的意思。它不仅仅是当前线程可以存取副本值，而且它的子线程也可以存取这个副本值。