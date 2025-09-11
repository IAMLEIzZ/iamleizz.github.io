---
title: Java 并发串讲
date: 2025-09-09T23:43:19+08:00
lastmod: 2025-09-09T23:43:19+08:00
draft: false
tags:
categories:
author: 小石堆
---
## <center>串讲内容</center>
从 Java 的并发的手段出发：
1. Synchorized
2. Volatile
3. ReentrantLock
4. 原子操作类

再到并发的底层
1. JMM
2. 内存可见性和有序性

再到常见并发工具类
1. ConcurrentHashMap & HashTable
2. JUC
## <center>并发安全手段</center>
Java 的并发一直是面试的考点，这里并发安全手段主要整理了 Synchorized 关键字、Volatile 关键字、ReentranLock、原子操作类这四个
### <center>Synchorized </center>
特性：
1. 支持可重入
2. 支持偏向锁、轻量级锁、重量级锁
3. 非公平
Sync 关键字主要作用于方法，作用是标记某个对象或类的某个方法，在同一时刻只能有一个线程进行操作。其主要用法先了解下：
```Java
//  1. 作用于代码块
synchorized(lock){
}
// 2. 作用于普通方法
public synchorized void xxxx(){
}
// 3. 作用于静态方法
public static synchorized void xxxx(){
}
```
上面的代码块，从上到下，Sync 关键字加锁的粒度也是逐渐增大，对于 1 号，锁的粒度是括号里的 lock；对于 2 号，锁的粒度是调用这个方法的对象；对于 3 号，锁的粒度是这个 Class。
#### Sync 实现原理
原理是**使用了JVM 内部的 Monitor 对象**。
	Monitor 对象：每个 **Java 的对象（包括类的 Class 对象）生来就与一个 Mointor 对象所绑定**，其是 JVM 内部的一种同步机制。Monitor **内部有一个锁**，来保证同一时刻只有一个线程进入访问。同时 Monitor 内部还维护了一个**等待队列**，用于实现 wait() & notify() 机制。
也就是说，Sync 是利用了 JVM 的机制，来实现的并发可靠，使用的时候无需手动 Lock & Unlock（区别于 ReentrantLock）。
Monitor 对象指针存在于 Java 对象的头部字段（MarkWord）中，每个对象都会生成自己的 MarkWord，其中包含了锁的状态、GC 相关标记、以及 Monitor 指针。
#### Sync 的锁升级
在 JDK1.7 之前，Sync 的锁只有重量级锁一种，底层是直接使用 OS 的 Mutex 操作的，首先这种方是虽然很大程度上保证了并发可靠，但是带来的消耗极大，因为每次加锁解锁都要与 OS 内核交互；因此，在 JDK 1.6 ，Sync 引入了三种锁，Jvm 会根据当前的系统并发情况，升级锁的强度。
无锁->偏向锁->轻量级锁->重量级锁
1. 偏向锁
	Java 的开发团队发现，在并发系统中，很多时候并不是线程交替获得锁，而是一个线程经常访问某个锁，因此，偏向锁诞生。
	偏向锁会在 MarkWord 中记录最近获得这个锁的线程 ID，如果下次获得这个锁的线程是同一个，则无需进行加锁操作，可以直接进入临界区
2. 轻量级锁
	在并发度不高的时候，例如所个线程交替获得锁，但不存在严重的竞争，此时，Jvm 会将锁升级为轻量级锁。
	轻量级锁是指：在线程进入临界区之前，不实用 OS 的互斥量，而是在当前线程栈中创建一个锁记录，然后通过 CAS 尝试让对象的 MarkWork 中的锁指向这个锁记录，从而达到上锁的目的。**CAS 成功，则获得锁**；**CAS 失败，则存在竞争，线程自旋尝试获得锁**。
3. 重量级锁
	锁竞争非常强烈的时候，例如当自旋次数过多还拿不到锁时，轻量级锁会膨胀为重量级锁。这时 JVM 会把线程阻塞，挂起到 OS 层的互斥量（mutex）上，等待唤醒。
![image.png](http://43.139.219.135:9000/blog-pic/images/20250910115829697.png)
### <center>ReentrantLock</center>
特性：
1. 可重入
2. 支持公平锁与非公平锁
3. 支持多路选择通知
4. 支持响应中断、超时、尝试获取锁
5. 需要手动 Lock 和 UnLock
RxxLock 底层依赖于 Java 的 AQS（AbstractQueueSynchronizer），下面先展示其基本用法
```Java
public class LockDemo {
	private final ReentrantLock lock = new ReentrantLock();
	
	public void demo1() {
		// 加锁
		lock.lock();
		try {
			// 临界区业务
		} finally {
			// 必须在 finally 中释放锁
			lock.unlock(); 
		}
	}
}
```
可以看出，首先 ReentrantLock 需要手动加锁解锁。lock 一定要在 try 之前，解锁一定要在 finally 中解锁。
#### AQS
AQS 是 Java 底层的一个抽象类，几乎所有 java.util.concurrent 包里的核心同步工具（锁、信号量、栅栏等）都是基于它构建的。其位于 `java.util.concurrent.locks` 下。
其**通过一个 volatile 修饰的 state 属性 + 一个等待队列以实现线程的排队和唤醒**。
其设计围绕两个点：
1. `private volatile int state`
	- 用 volatile int state 表示资源的占用情况。
	- 子类通过实现 tryAcquire()、tryRelease() 来定义“如何获取/释放资源”。
	- 修改 state 的时候用 **CAS（Compare-And-Swap）** 保证原子性。
2. `private transient volatile Node head 和 private transient volatile Node tail`
	- 当线程获取资源失败时，AQS 会把它放入一个 **CLH（双向链表）队列**。
	- 当资源释放时，AQS 会从队列里唤醒下一个等待的线程。
#### ReentrantLock 实现原理
RxxLock 底层就是基于 AQS 的，与上面的原理类似
- 当线程获取到锁后，会将 state + 1
- 因为 RxxLock 是可重入的，所以当同一个线程获取锁的时候，state ++，不是同一个线程获取锁，则加入到等待队列中
- 当线程调用 unlock，state --，当 state = 0 时，释放锁，唤醒等待队列中的线程
注意，RxxLock 默认创建的是非公平的锁，即唤醒的不一定是最早进入队列的线程
#### Synchorized 和 ReentrantLock

|       | Synchorized   | ReentrantLock           |
| ----- | ------------- | ----------------------- |
| 底层    | Jvm 的 Monitor | Java 的 AQS 抽象类          |
| 是否可重入 | 可重入           | 可重入                     |
| 公平性   | 非公平           | 公平 & 非公平                |
| 灵活性   | 不灵活           | 支持响应中断、超时等获取锁；还支持多路选择通知 |
| 释放锁   | 自动释放          | 必须配合 unlock 手动解锁        |
### <center>Volatile 关键字</center>
上面的Sync 和 RxxLock 实际上是更高一层的，他们都支持了并发安全，也就是说能保证多线程在运作的时候，提供了**内存可见性**之外，还提供了**互斥机制**。而 vloatile 关键字则处于更底层，它的作用相对“轻量级”，其实际上保障的是并发时候的内存可见性和有序性。

	内存可见性：当一个线程修改了某个变量的值后，其他线程能立刻读到新值，保证上面这点，即保证了内存可见性
	有序性：代码可能会被编译器指令重排优化，在多线程环境下，防止某些指令被重新排列，即做到了有序性

由于现代计算机多级缓存的结构，很多时候线程访问变量都是优先在寄存器或者缓存中获取，而非主存。这就导致了无法保证内存可见性。对于单线程来讲，内存可见性无关紧要，但是多线程下，内存可见性就很重要了。
举个栗子：
```Java
class FlagTest {
    private static boolean flag = true;
    
    public static void main(String[] args) throws InterruptedException {
        new Thread(() -> {
            while (flag) { // 线程 A 一直在工作内存里读 flag
            }
            System.out.println("线程 A 结束");
        }).start();
        
        Thread.sleep(1000);
        flag = false; // 主线程修改了 flag
    }
}
```
在 1s 后，主线程将 flag 修改后，理论上我们期待线程 A 结束，但是因为 A 一直在自己的工作内存忠读取 flag，所以导致 A 永远无法跳出循环。
而上面的代码也很好修改，只需要让 flag 变成 volatile 类型就行，这样每次读写这个变量，线程都会直接访问主存。
#### Volatile 如何保障内存可见性和有序性
可见性：

	写操作直接将变量写入主存，读操作直接从主存中读
有序性：

	volatile 关键字通过内存屏障来禁止指令重排序
	- 在写 volatile 变量时，JMM 会插入 StoreStore 和 StoreLoad 屏障，保证该写操作之前的所有普通写操作一定先执行，后面的普通写操作不能提前
	- 在读 volatile 变量时，JMM 会插入 LoadLoad 和 LoadStore 屏障，保证该读操作之前的读操作不能延迟，该读操作之后的读操作不能提前

#### Volatile 不保证原子性
虽然说 volatile 关键字保证了内存可见性和有序性，但是并不保证原子性，例如对于 `i ++` 这一指令，先读入 i，再将 i + 1，再写回内存，这三个步骤，volatile 关键字能保证他们不乱序。但是无法把这三个步骤原子化，所以，Volatile 并不能保证并发安全。想要原子性，还是得 Sync 关键字和锁等并发手段。
### <center>原子操作类</center>
![image.png](http://43.139.219.135:9000/blog-pic/images/20250910172733879.png)
## JMM
Java 内存模型（Java Memory Model，JMM），其定义了Java 程序中的变量、线程如何和主存以及工作内存进行交互的规则。会涉及到指令重排序和内存可见性等问题，所以并发这里会涉及到 JMM。
JMM：

	JMM 是 Java 为了保证多线程并发时并发安全，提出的一套线程访问主存和工作内存的规则。其主要涉及到的是内存可见性、有序性以及原子性等。

JAVA运行时内存区域：

	Java 运行时内存区域指的是 Java 程序在运行时，会将内存分为，堆、方法区、虚拟机栈、方法栈以及程序计数器等。描述的是 Java 程序在运行的时候，内存区域的逻辑划分。

所以不要将二者混为一谈啦！
## 并发工具类
### <center>ConcurrentHashMap & Hashtable</center>
想要程序并发操作 Map，则需要用使用这两个并发安全的 Map，二者在底层实现上有巨大差异，在 JDK1.8 后，ConcurrentHashMap 性能很不错。
#### ConcurrentHashMap
CxxMap 底层数据结构类似 HashMap 的实现，1.8 后采用的是 数组 + 链表 + 红黑树的解决方案。当链表长度大于 8 时，会考虑是否转化为红黑树（这里扩容原则类似HashMap）。
JDK1.8 之前
![image.png](http://43.139.219.135:9000/blog-pic/images/20250910225240026.png)

JDK1.8 之后
![image.png](http://43.139.219.135:9000/blog-pic/images/20250910225255828.png)
上面两张图，也透露出了 ConcurrentHashMap 的加锁原则。当两个线程同时访问到一个桶的时候，则对桶加锁，而 1.7 是基于段的，所以加锁的粒度会更大，有损性能。
其中，1.7 的 Segment 锁实际上是基于 ReentrantLock 的，而 1.8 的 Node 锁是基于 CAS + Synchorized 的。
#### HashTable
HashTable 内部方法都经过 Synchorized 关键字修饰，因此也是并发安全的数据结构。其数据结构与 1.7 的 HashMap 类似。而区别就在 HashTable 锁的粒度是整个 Table，这就导致多线程同时访问 Table 的时候，即使访问的不是同一个 Key，也无法并行，效率低下。
![image.png](http://43.139.219.135:9000/blog-pic/images/20250910230101571.png)

### JUC
JDK 中关于并发的类大多都在 JUC（java.utils.concurrent）包下。
#### Semaphore
Sema 是一个计数信号量，作用是限制可以访问某个资源的线程数目。Sema 主要有两个方法，`acquire()` 和 `release()`。`acquire()` 会尝试获取一个 sema，如果获取不到线程会进入阻塞态。
**Sema 只是控制同时访问某个特定资源的操作数量，但他无法保证并发安全，因此并发安全还是要通过并发控制手段。也就是说，获取到 Sema 后，后续的业务逻辑要保证并发安全要自己控制。**
```Java
public class CounterTest {
    private static int count = 0;
    private static Semaphore semaphore = new Semaphore(3);

    public static void main(String[] args) {
        Runnable task = () -> {
            try {
                semaphore.acquire();
                // 临界区
                count++;  // 这里不是线程安全的
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                semaphore.release();
            }
        };

        for (int i = 0; i < 1000; i++) {
            new Thread(task).start();
        }
    }
}
```
#### CountDownLatch
CDLatch 是用来控制资源启动线程的，他允许一个或多个线程等待，知道其他线程指定完后执行。CountDownLatch 有一个计数器，可以通过 `countDown()` 方法对计数器的数目进行减一操作，也可以通过 `await()` 方法来阻塞当前线程，直到计数器的值为 0。
核心方法：
- **new CountDownLatch(int count)**：初始化时指定计数值。
- **await()**：调用的线程在这里等待，直到计数为 0 才继续执行。
- **countDown()**：让计数器减一。
```Java
import java.util.concurrent.CountDownLatch;

public class RaceDemo {
    public static void main(String[] args) throws InterruptedException {
        int players = 3;
        CountDownLatch latch = new CountDownLatch(players);

        for (int i = 1; i <= players; i++) {
            new Thread(() -> {
                System.out.println(Thread.currentThread().getName() + " 准备好了");
                latch.countDown();  // 每个线程准备好就 countDown
            }, "运动员-" + i).start();
        }

        latch.await(); // 等待所有运动员准备好
        System.out.println("裁判：所有人准备好了，比赛开始！");
    }
}
```
#### CyclicBarrier
CyBarrier 与 CDLatch 作用类似，但是 CyBarrier 可以复用。
核心方法：
- **new CyclicBarrier(int parties)**：指定参与线程的数量。
- **await()**：线程调用此方法表示“我到达集合点”，然后阻塞等待其他线程。直到所有线程都调用了 await()，屏障才会打开，所有线程继续运行。
- **CyclicBarrier(int parties, Runnable barrierAction)**：带一个任务参数，所有线程到齐后，优先执行这个任务，然后大家再继续。
CDLatch 用完就废了，而 CyBarrier 可以复用，例如多关游戏，每关都需要玩家准备好才能开始。这时候每关都创建一个 CDLatch 就没必要了。
```Java
import java.util.concurrent.CyclicBarrier;

public class CyclicBarrierLoopDemo {
    public static void main(String[] args) {
        int players = 3;
        CyclicBarrier barrier = new CyclicBarrier(players, () -> {
            System.out.println("本关结束，准备进入下一关");
        });

        for (int i = 1; i <= players; i++) {
            final int id = i;
            new Thread(() -> {
                try {
                    for (int round = 1; round <= 2; round++) {
                        System.out.println("玩家 " + id + " 完成第 " + round + " 关");
                        barrier.await(); // 等所有人完成再进入下一关
                    }
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }).start();
        }
    }
}
```

| **特性**   | **CountDownLatch** | **CyclicBarrier** |
| -------- | ------------------ | ----------------- |
| **重用性**  | 一次性                | 可重用（循环）           |
| **计数方向** | 倒数到 0 后触发          | 正数到齐后触发           |
| **等待对象** | 主线程等子线程 / 若干线程等事件  | 多个线程相互等待          |
| **典型场景** | 系统初始化、主线程等待子任务完成   | 多个线程分阶段同步（如关卡制）   |