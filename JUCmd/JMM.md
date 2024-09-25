# JMM

JMM 即 Java Memory Model，它定义了主存、工作内存抽象概念，底层对应着 CPU 寄存器、缓存、硬件内存、CPU 指令优化等

JMM 体现在以下几个方面

+ 原子性

  保证指令不会受到线程上下文切换的影响 

+ 可见性

  保证指令不会受 CPU 缓存的影响

+ 有序性

  保证指令不会受 CPU 指令并行优化的影响



## 可见性

如下代码：

```java
static boolean run = true;
public static void main(String[] args) throws InterruptedException {
    Thread t1 = new Thread(() -> {
        while (run) {

        }
    });
    t1.start();
    Thread.sleep(1000);
    log.debug("停止");
    run = false;
}
```

并没有停下来

分析一下：

1. 初始状态，t 线程刚开始从主内存读取了 run 的值到工作内存

   ![image-20240925095722217](https://s2.loli.net/2024/09/25/RNWJLMjQUBaI4OF.png)

2. 因为 t 线程频繁从主内存中读取 run 的值，JIT 编译器会将 run 的值缓存至自己工作内存中的高速缓存中，减少对主存中 run 的访问，提高效率

   ![image-20240925095855296](https://s2.loli.net/2024/09/25/Ab2B3rEFDg4SIes.png)

3. **1 秒后**，main 线程修改了 run 的值，并同步至主存，而 t 是从自己工作内存中的高速缓存中读取这个变量的值，结果永远是**旧的值**

   ![image-20240925100027898](https://s2.loli.net/2024/09/25/ZYPCigB9qSkc1Eu.png)

**解决方式**

1. 使用**`volatile`**关键字修饰共享的变量

   ```java
   volatile static boolean run = true;
   ```

2. 使用**`synchronized`**将共享变量放在同步块内

   ```java
   static final Object lock = new Object();
   
   public static void main(String[] args) throws InterruptedException {
       Thread t1 = new Thread(() -> {
           while (run) {
               synchronized (lock) {
                   if (!run) {
                       break;
                   }
               }
           }
       });
       t1.start();
       Thread.sleep(1000);
       log.debug("停止");
       synchronized (lock) {
           run = false;
       }
   }
   ```

**区别**：

+ 使用`synchronized`需要创建monitor，属于重量级操作，而使用`volatile`属于轻量
+ 在可见性层面推荐使用`volatile`



## 可见性 VS 原子性

`volatile`适用于一个写线程和多个读线程的情况

从字节码理解是这样的：

```java
getstatic      run  // 线程 t 获取 run  true
getstatic      run  // 线程 t 获取 run  true
getstatic      run  // 线程 t 获取 run  true
getstatic      run  // 线程 t 获取 run  true
putstatic      run  // 线程 main 修改 run 为 false，仅此一次
getstatic      run  // 线程 t 获取 run  false
```

比较一下之前在线程安全讲到的例子：两个线程一个 `i++`，一个`i--`，只能保证获取到最新的值，不能解决指令交错

```java
getstatic      i  // 线程 2 获取 i  线程内 i = 0
getstatic      i  // 线程 1 获取 i  线程内 i = 0
iconst_1          // 线程 1 准备常量 1
iadd              // 线程 1 自增 线程内 i = 1
putstatic      i  // 线程 1 将 i = 1 存入静态变量 i = 1
iconst_1          // 线程 2 准备常量 1 
isub              // 线程 2 自减 线程内 i = -1
putstatic      i  // 线程 2 将修改后的值存入静态变量 i = -1
```

> `synchronized`语句块既**可以保证代码块内的原子性**，也同时**保证代码块内变量的可见性**。但缺点是`synchronized`是属于重量级操作，性能相对较低



## 改进两阶段终止模式

```java
@Slf4j
public class Test1 {
    public static void main(String[] args) throws InterruptedException {
        TwoPhaseTermination tpt = new TwoPhaseTermination();
        tpt.start();

        Thread.sleep(2000);
        tpt.stop();
    }
}

@Slf4j
class TwoPhaseTermination {
    private Thread monitor;
    volatile private boolean stop;

    public void start() {
        monitor = new Thread(() -> {
            while (true) {
                if (stop) {
                    log.debug("料理后事");
                    break;
                }

                try {
                    Thread.sleep(500);
                    log.debug("执行监控");
                } catch (InterruptedException e) {
                }
            }
        });
        monitor.start();
    }

    public void stop() {
        stop = true;
    }
}
```



## Balking 模式

Balking （犹豫）模式用在一个线程发现另一个线程或本线程已经做了某件相同的事，那么本线程无需再做，直接结束放回

```java
@Slf4j
public class Test1 {
    public static void main(String[] args) throws InterruptedException {
        TwoPhaseTermination tpt = new TwoPhaseTermination();
        tpt.start();
        tpt.start();
        tpt.start();

        Thread.sleep(2000);
        tpt.stop();
    }
}

@Slf4j
class TwoPhaseTermination {
    private Thread monitor;
    volatile private boolean stop;

    private boolean starting = false;

    public void start() {
        /* 这里就是 Balking 模式的代码 */
        synchronized (this) {				
            if (starting) return;
            starting = true;
        }
        

        monitor = new Thread(() -> {
            while (true) {
                if (stop) {
                    log.debug("料理后事");
                    break;
                }

                try {
                    Thread.sleep(500);
                    log.debug("执行监控");
                } catch (InterruptedException e) {
                }
            }
        });
        monitor.start();
    }

    public void stop() {
        stop = true;
    }
}
```

 

## 有序性

JVM 会在不影响正确性的前提下，可以调整语句的执行顺序，

**使用关键字`volatile`可以保证执行的有序性，防止代码重排序**



## volatile 原理

volatile 的底层实现原理是内存屏障，Memory Barrier （Memory Fence）

+ 对 volatile 变量的写指令后会加入写屏障
+ 对 volatile 变量的读指令前会加入读屏障

### 如何保证可见性

+ 写屏障保证在该屏障之前的，对共享变量的改动，都同步到主存当中

  ```java
  public void actor2(I_Result r) {
      num = 2;
      ready = true; // ready 是 volatile 修饰带写屏障
  }
  ```

+ 而读屏障保证在该屏障之后，对共享变量的读取，加载的是主存中最新数据

  ```java
  public void actor1(I_Result r) {
      // ready 是 volatile 修饰带读屏障
      if (ready) {
          r.r1 = num + num;
      } else {
          r.r1 = 1;
      }
  }
  ```

### 如何保证有序性

+ 写屏障会确保指令重排序时，不会将写屏障之前的代码排到写屏障之后

  ```java
  public void actor2(I_Result r) {
      num = 2;
      ready = true; // ready 是 volatile 修饰带写屏障
  }
  ```

+ 读屏障会确保指令重排序时，不会将读屏障之后的代码排在读屏障之前

  ```java
  public void actor1(I_Result r) {
      // ready 是 volatile 修饰带读屏障
      if (ready) {
          r.r1 = num + num;
      } else {
          r.r1 = 1;
      }
  }
  ```

**读写屏障**并不能解决指令交错：

+ 写屏障仅仅保证之后的读能够读到最新的结果，但不能保证读跑到他前面去

+ 而有序性的保证也只是保证了本线程内相关代码不被重排序

  ![image-20240925123441617](https://s2.loli.net/2024/09/25/8QgI27B5fHehpo6.png)

### double-checked locking 问题

```java
public final class Singleton {
    private Singleton() {}
    private static Singleton INSTANCE = null;
    public static Singleton getInstance() {
        if (INSTANCE == null) {
            synchronized(Singleton.class) {
                if (INSTANCE == null) {
                    INSTANCE = new Singleton();
                }
            }
        }
        return INSTANCE;
    }
}
```

以上的实现特点：

+ 懒惰实例化
+ 首次使用`getInstance()`才使用`synchronized`加锁，后续使用时无需加锁
+ 有隐含的，第一个 if 使用了 `INSTANCE`变量，是在同步块之外

**但是，在多线程情况下，上面代码存在问题**

```java 
 0 getstatic #2 <org/example/test/Singleton.INSTANCE : Lorg/example/test/Singleton;>
 3 ifnonnull 37 (+34)
 6 ldc #3 <org/example/test/Singleton>
 8 dup
 9 astore_0
10 monitorenter
11 getstatic #2 <org/example/test/Singleton.INSTANCE : Lorg/example/test/Singleton;>
14 ifnonnull 27 (+13)
17 new #3 <org/example/test/Singleton>
20 dup
21 invokespecial #4 <org/example/test/Singleton.<init> : ()V>
24 putstatic #2 <org/example/test/Singleton.INSTANCE : Lorg/example/test/Singleton;>
27 aload_0
28 monitorexit
29 goto 37 (+8)
32 astore_1
33 aload_0
34 monitorexit
35 aload_1
36 athrow
37 getstatic #2 <org/example/test/Singleton.INSTANCE : Lorg/example/test/Singleton;>
40 areturn
```

![image-20240925130015170](https://s2.loli.net/2024/09/25/WenGZFcp14MRNgs.png)

### 解决方法

`private static Singleton INSTANCE = null;`

改为

`private volatile static Singleton INSTANCE = null;`**加入了读写屏障**

```java
// 加入对 INSTANCE 变量的读屏障
 0 getstatic #2 <org/example/test/Singleton.INSTANCE : Lorg/example/test/Singleton;>
 3 ifnonnull 37 (+34)
 6 ldc #3 <org/example/test/Singleton>
 8 dup
 9 astore_0
10 monitorenter
11 getstatic #2 <org/example/test/Singleton.INSTANCE : Lorg/example/test/Singleton;>
14 ifnonnull 27 (+13)
17 new #3 <org/example/test/Singleton>
20 dup
21 invokespecial #4 <org/example/test/Singleton.<init> : ()V>
24 putstatic #2 <org/example/test/Singleton.INSTANCE : Lorg/example/test/Singleton;>
// 加入对 INSTANCE 变量的写屏障
27 aload_0
28 monitorexit
29 goto 37 (+8)
32 astore_1
33 aload_0
34 monitorexit
35 aload_1
36 athrow
37 getstatic #2 <org/example/test/Singleton.INSTANCE : Lorg/example/test/Singleton;>
40 areturn
```

![image-20240925130420337](C:/Users/Kk/AppData/Roaming/Typora/typora-user-images/image-20240925130420337.png)

**最终正确写法**

```java
public final class Singleton {
    private Singleton() {}
    private volatile static Singleton INSTANCE = null;
    public static Singleton getInstance() {
        if (INSTANCE == null) {
            synchronized(Singleton.class) {
                if (INSTANCE == null) {
                    INSTANCE = new Singleton();
                }
            }
        }
        return INSTANCE;
    }
}
```

### Happens-Before 

规定了对共享变量的写操作对其他线程的读操作可见

+ 线程解锁 m 之前对变量的写，对于接下来对 m 加锁的其他线程对该变量的读可见

  ```java
  static int x;
  static Object m = new Object();
  new Thread(() -> {
      synchronized(m) {
          x = 10;
      }
  }).start();
  new Thread(() -> {
      synchronized(m) {
          System.out.println(x);
      }
  }).start();
  ```
  
+ 线程对 volatile 变量的写，对接下来其他线程对该变量的读可见

  ```java
  volatile static int x;
  new Thread(() -> {
      x = 10;
  }).start();
  
  new Thread(() -> {
      System.out.println(x);
  }).start();
  ```

+ 线程 start 前对变量的写，对该线程开始后对该变量的读可见

  ```java
  static int x;
  x = 10;
  new Thread(() -> {
      System.out.println(x);
  }).start();
  ```

+ 线程结束前对变量的写，对其他线程得知它结束后的读可见

  ```java
  static int x;
  Thread t1 = new Thread(() -> {
      x = 10;
  }).start();
  
  t1.join();
  System.out.println(x);
  ```

+ 线程 t1 打断 t2 前对变量的写，对于其他线程得知 t2 被打断后对变量的读可见

  ```java
  static int x;
  
  public static void main(String[] args) {
      Thread t2 = new Thread(() -> {
          while (true) {
              if (Thread.currentThread().isInterrupted()) {
                  System.out.println(x);
                  break;
              }
          }
      }).start();
      
      new Thread(() -> {
          Thread.sleep(1000);
          x = 10;
          t2.interrupt();
      }).start();
      
      while (!t2.isInterrupted()) {
          Thread.yield();
      }
      System.out.println(x);
  }
  ```

+ 对变量默认值（0，false，null）的写，其他线程对该变量的读可见

+ 具有传递性，如果 x  `hb`  y，y `hb` z，那么有x `hb` z，配合 volatile 的防指令重排

  ```java
  volatile static int x;
  static int y;
  
  new Thread(() -> {
      y = 10;
      x = 20;
  }).start();
  
  new Thread(() -> {
      System.out.println(x);
      // x=20 对 t2 可见，同时 y=10 也对 t2可见
  }).sta
  ```

  
