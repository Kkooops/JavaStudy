# Java线程



## 创建和允许线程

### 直接使用Thread

```java
Thread t = new Thread() {
    @Override
    public void run() {
        // 要执行的代码
    }
}
Thread t = new Thread(() -> { /* 要执行的代码 */ });
t.start();
```



### 使用Runnable 配合 Thread

用Runnable更容易与线程池等高级API配合

用Runnable更灵活

```java
Runnable runnable = new Runnable() {
    @Override
    public void run() {
        // 要执行的代码
    }
};

//Lambda简化
Runnable runnable1 = () -> { /* 要执行的代码*/ };

new Thread(runnable).start();
```



### FutureTask 配合 Thread

FutureTask能够配合Callable类型的参数，用来处理有返回结果的情况

```java
 FutureTask<Integer> task = new FutureTask<>(new Callable<Integer>() {
    @Override
    public Integer call() throws Exception {
        log.debug("running...");
        Thread.sleep(2000);
        return 100;
    }
});
Thread thread = new Thread(task);
thread.start();

// 使用get()获取返回结果，当前主线程会阻塞
log.info("{}", task.get());
```



## 查看进程线程的方法

### WIndows命令

`tasklist` 查看进程

`taskkill` /F /PID `PID` 杀死进程

### Linux命令

`ps -ef` 查看所有进程

`ps -fT -p <PID>` 查看某个进程的所有线程

`kill` 杀死进程

`top`按大写`H`切换是否显示线程

`top -H -p <PID>` 查看某个进程的所有线程



## 线程上下文切换 （Thread Context Switch）

因为以下原因导致CPU不再执行当前的线程，转而执行另一个线程的代码

+ 线程CPU时间片用完
+ 垃圾回收
+ 有更高优先级线程需要运行
+ 线程自己调用了sleep、yield、wait、join、park、synchronized、lock等方法

当Context Switch发生时，需要由操作系统保存当前线程状态，并恢复另一个线程的状态

+ 状态包括**程序计数器**、虚拟机栈中每个栈帧的信息，如**局部变量**、**操作数栈**、**返回地址**等
+ 频繁上下文切换会影响性能



## start与run

直接调用thread.run()方法并不会创建一个新的线程，所以必须使用start()

## sleep与yield

### sleep

1. 调用sleep会让线程状态从Running进入Timed Waiting状态
2. 其他线程可以使用interrupt方法打断正在睡眠的线程，这时sleep方法会抛出InterruptedException
3. 睡眠结束后的线程未必会立刻得到执行
4. 建议用 TimeUnit 的 sleep 代替 Thread 的sleep来获得更好的可读性

### yield

1. 调用yield会让当前线程从Running进入Runnable就绪状态，然后调度执行其他线程
2. 具体的实现依赖于操作系统的任务调度器

## Join方法详解

> 等待线程运行结束

需要等待结果返回，才能继续运行就是**同步**

不需要等待结果返回，就能继续运行就是**异步**

join方法也可以设置最大等待时间

## interrupt方法详解

### 打断 sleep, wait, join的线程

```java
Thread t = new Thread(() -> {
    try {
        Thread.sleep(2000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
});
t.start();

t.interrupt();
Thread.sleep(100);
log.info("打断状态：{}", t.isInterrupted());

// sleep，join会把打断标记清除
// 输出：
java.lang.InterruptedException: sleep interrupted
	at java.lang.Thread.sleep(Native Method)
	at org.example.test.Test1.lambda$main$0(Test1.java:15)
	at java.lang.Thread.run(Thread.java:750)
2024-09-20 13:56:39 [main] INFO  org.example.test.Test1 - 打断状态：false
```

### 打断正常运行的线程

```java
public static void main(String[] args) throws ExecutionException, InterruptedException {
    Thread t = new Thread(() -> {
        while (true) {
            log.info("running");
            if (Thread.currentThread().isInterrupted())
                break;
        }
    });
    t.start();
    t.interrupt();
    log.info("打断状态：{}", t.isInterrupted());
}
// 输出：
2024-09-20 14:03:13 [Thread-0] INFO  org.example.test.Test1 - running
2024-09-20 14:03:13 [main] INFO  org.example.test.Test1 - 打断状态：true

// 线程完成并清除其中断状态所以
public static void main(String[] args) throws ExecutionException, InterruptedException {
    Thread t = new Thread(() -> {
        while (true) {
            log.info("running");
            if (Thread.currentThread().isInterrupted())
                break;
        }
    });
    t.start();
    t.interrupt();
    Thread.sleep(1000);       // 等待线程结束
    log.info("打断状态：{}", t.isInterrupted());
}
// 输出：
2024-09-20 14:04:57 [Thread-0] INFO  org.example.test.Test1 - running
2024-09-20 14:04:59 [main] INFO  org.example.test.Test1 - 打断状态：false

```

### **两阶段终止模式**

> 在一个线程T1中如何“优雅”终止线程T2 ？这里的优雅指的是给T2一个料理后事的机会

错误思路：

+ 使用线程对象的`stop()`方法

  可能导致无法释放资源，导致死锁

+ 使用`System.exit(int)`方法

  导致线程所在的进程停止，进程中所有线程都会被停止

![image-20240920142222236](https://s2.loli.net/2024/09/20/nLDNfsKmI5FUbGr.png)

```java
@Slf4j
public class Test1 {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        TwoPhaseTermination tpt = new TwoPhaseTermination();
        tpt.start();

        Thread.sleep(3000);
        tpt.stop();
    }
}

@Slf4j
class TwoPhaseTermination {
    private Thread monitor;

    public void start() {
        monitor = new Thread(() -> {
            while (true) {
                Thread cur = Thread.currentThread();
                if (cur.isInterrupted()) {
                    log.debug("料理后事");
                    break;
                }

                try {
                    Thread.sleep(1000);
                    log.debug("执行监控");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                    cur.interrupt();
                }
            }
        });
        monitor.start();
    }

    public void stop() {
        monitor.interrupt();
    }
}
//输出：
2024-09-20 14:22:35 [Thread-0] DEBUG org.example.test.TwoPhaseTermination - 执行监控
2024-09-20 14:22:36 [Thread-0] DEBUG org.example.test.TwoPhaseTermination - 执行监控
java.lang.InterruptedException: sleep interrupted
	at java.lang.Thread.sleep(Native Method)
	at org.example.test.TwoPhaseTermination.lambda$start$0(Test1.java:35)
	at java.lang.Thread.run(Thread.java:750)
2024-09-20 14:22:37 [Thread-0] DEBUG org.example.test.TwoPhaseTermination - 料理后事
```

### 打断park线程

打断park线程，不会清空打断状态

```java
public static void main(String[] args) throws ExecutionException, InterruptedException {
    Thread t1 = new Thread(() -> {
        log.debug("park...");
        LockSupport.park();
        log.debug("unpark...");
        log.debug("打断状态：{}", Thread.currentThread().isInterrupted());
    });
    t1.start();

}
// 输出：
2024-09-20 14:32:44 [Thread-0] DEBUG org.example.test.Test1 - park...
并且程序没有停止

public static void main(String[] args) throws ExecutionException, InterruptedException {
    Thread t1 = new Thread(() -> {
        log.debug("park...");
        LockSupport.park();
        log.debug("unpark...");
        log.debug("打断状态：{}", Thread.currentThread().isInterrupted());
    });
    t1.start();

    Thread.sleep(1000);
    t1.interrupt();
}
// 输出：
2024-09-20 14:33:16 [Thread-0] DEBUG org.example.test.Test1 - park...
2024-09-20 14:33:17 [Thread-0] DEBUG org.example.test.Test1 - unpark...
2024-09-20 14:33:17 [Thread-0] DEBUG org.example.test.Test1 - 打断状态：true

进程已结束，退出代码为 0
```

park()在打断标记为true的时候会失效

```java
public static void main(String[] args) throws ExecutionException, InterruptedException {
        Thread t1 = new Thread(() -> {
            log.debug("park...");
            LockSupport.park();
            log.debug("unpark...");
            log.debug("打断状态：{}", Thread.currentThread().isInterrupted());
            LoSupport.park();
            log.debug("hello");
        });
        t1.start();

        Thread.sleep(1000);
        t1.interrupt();
    }
// 输出：
2024-09-20 14:33:48 [Thread-0] DEBUG org.example.test.Test1 - park...
2024-09-20 14:33:49 [Thread-0] DEBUG org.example.test.Test1 - unpark...
2024-09-20 14:33:49 [Thread-0] DEBUG org.example.test.Test1 - 打断状态：true
2024-09-20 14:33:49 [Thread-0] DEBUG org.example.test.Test1 - hello

进程已结束，退出代码为 0
```

如果想要park() 再次生效，需要设置打断标记为false, 可以将上面代码中`Thread.currentThread().isInterrupted()`替换为`Thread.interrupted()`

## 不推荐的方法

`stop()`、`suspend()`、`resume()`容易导致资源得不到释放，不推荐使用



## 主线程与守护线程

默认情况下，Java进程需要等待所有线程都运行结束，才会结束。

有一种特殊的线程叫做**守护线程**，只要其他非守护线程都结束了，即使守护线程的代码没有运行结束，也会强制结束

```java
@Slf4j
public class Test1 {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        Thread thread = new Thread(() -> {
            while (true) {
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
                log.info("守护线程在运行");
            }
        });
        thread.setDaemon(true);
        thread.start();

        Thread.sleep(2000);
        log.info("Main结束");
    }
}
// 输出
2024-09-20 14:42:34 [Thread-0] INFO  org.example.test.Test1 - 守护线程在运行
2024-09-20 14:42:35 [main] INFO  org.example.test.Test1 - Main结束
2024-09-20 14:42:35 [Thread-0] INFO  org.example.test.Test1 - 守护线程在运行

进程已结束，退出代码为 0
```

**垃圾回收器线程**就是一种守护线程

## 五种状态

这是从**操作系统**层面来描述的

![image-20240920144538012](https://s2.loli.net/2024/09/20/ocMNh5aDfAeiBTk.png)

+ 初始状态

  仅在语言层面创建了线程对象，还未与OS线程关联

+ 可运行（就绪）状态

  已经被创建（与OS线程关联），可以由CPU进行调度执行

+ 运行状态

  获取了CPU时间片运行中的状态

  + 当CPU时间片用完时，会从**运行状态**转换为**可运行状态**，会导致线程的上下文切换

+ 阻塞状态

  + IO操作等阻塞API会进入阻塞状态
  + 等IO操作完毕，会由OS唤醒，转换至**可运行状态**

+ 终止状态

  标识线程已经执行完毕，生命周期结束，不会再转换为其他状态

## 六种状态

这是从**Java API** 层面来描述的

这是根据`Thread.State`枚举，分为6种状态

![image-20240920145159716](https://s2.loli.net/2024/09/20/hPAvLieU4j5G1E3.png)

+ NEW

  线程刚被创建，还没有执行`start()`方法

+ RUNNABLE

  调用了`start()`方法之后。注意，Java API层面的`RUNNABLE`状态涵盖了OS层面的**可运行状态**、**运行状态**、**阻塞状态**

+ BLOCKED

  线程正在等待获取一个同步锁

+ WAITING

  线程正在无限期等待另一个线程执行特定操作

+ TIMED_WAITING

  线程正在等待另一个线程执行操作，但有时间限制
  
+ TERMINATED

  线程终止
