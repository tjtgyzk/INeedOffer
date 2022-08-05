# LockSupport

## 简介

- LockSupport用来**创建锁**和**其他同步类**的**基本线程阻塞原语**。
- 简而言之，当调用`LockSupport.park()`时，表示**当前线程将会等待，直至获得许可**，当调用`LockSupport.unpark()`时，必须把**等待获得许可的线程**作为**参数**进行传递，好**让此线程继续运行**。

## 源码分析

### 类的属性

```java
public class LockSupport {
    // Hotspot implementation via intrinsics API
    private static final sun.misc.Unsafe UNSAFE;
    // 表示内存偏移地址
    private static final long parkBlockerOffset;
    // 表示内存偏移地址
    private static final long SEED;
    // 表示内存偏移地址
    private static final long PROBE;
    // 表示内存偏移地址
    private static final long SECONDARY;
    
    static {
        try {
            // 获取Unsafe实例
            UNSAFE = sun.misc.Unsafe.getUnsafe();
            // 线程类类型
            Class<?> tk = Thread.class;
            // 获取Thread的parkBlocker字段的内存偏移地址
            parkBlockerOffset = UNSAFE.objectFieldOffset
                (tk.getDeclaredField("parkBlocker"));
            // 获取Thread的threadLocalRandomSeed字段的内存偏移地址
            SEED = UNSAFE.objectFieldOffset
                (tk.getDeclaredField("threadLocalRandomSeed"));
            // 获取Thread的threadLocalRandomProbe字段的内存偏移地址
            PROBE = UNSAFE.objectFieldOffset
                (tk.getDeclaredField("threadLocalRandomProbe"));
            // 获取Thread的threadLocalRandomSecondarySeed字段的内存偏移地址
            SECONDARY = UNSAFE.objectFieldOffset
                (tk.getDeclaredField("threadLocalRandomSecondarySeed"));
        } catch (Exception ex) { throw new Error(ex); }
    }
}
  
```

### 构造函数

- **仅有一个私有的空构造函数，无法被实例化**

### 核心函数

在分析LockSupport函数之前，先引入s**un.misc.Unsafe类**中的park和unpark函数，因为LockSupport的核心函数都是基于Unsafe类中定义的park和unpark函数，下面给出两个函数的定义:

```java
public native void park(boolean isAbsolute, long time);
public native void unpark(Thread thread);
```

对两个函数的说明如下:

- park函数，阻塞线程，并且该线程在下列情况发生之前都会被阻塞: 
  - 调用unpark函数，释放该线程的许可。
  - 该线程被中断。
  - 设置的时间到了。并且，当time为绝对时间时，isAbsolute为true，否则，isAbsolute为false。当time为0时，表示无限等待，直到unpark发生。
- unpark函数，释放线程的许可，即激活调用park后阻塞的线程。这个函数不是安全的，调用这个函数时要确保线程依旧存活。

#### park

```java
public static void park();
public static void park(Object blocker);


//具体实现
public static void park() {
    // 获取许可，设置时间为无限长，直到可以获取许可
    UNSAFE.park(false, 0L);
}

public static void park(Object blocker) {
    // 获取当前线程
    Thread t = Thread.currentThread();
    // 设置Blocker
    setBlocker(t, blocker);
    // 获取许可
    UNSAFE.park(false, 0L);
    // 重新可运行后再此设置Blocker
    setBlocker(t, null);
}
```

这两个函数的区别仅在于park函数有没有blocker，即有没有设置线程的parkBlocker字段。

> 线程对象 Thread 里面有一个重要的属性 parkBlocker，它保存当前线程因为什么而 park。就好比停车场上停了很多车，这些车主都是来参加一场拍卖会的，等拍下自己想要的物品后，就把车开走。那么这里的 parkBlocker 大约就是指这场「拍卖会」。它是一系列冲突线程的管理者协调者，哪个线程该休眠该唤醒都是由它来控制的。
>
> 当线程被 unpark 唤醒后，这个属性会被置为 null。Unsafe.park 和 unpark 并不会帮我们设置 parkBlocker 属性，负责管理这个属性的工具类是 LockSupport，它对 Unsafe 这两个方法进行了简单的包装。

调用了park函数后，会禁用当前线程，除非许可可用。在以下三种情况之一发生之前，当前线程都将处于休眠状态，即**下列情况发生时，当前线程会获取许可，可以继续运行**。

- 其他某个线程将当前线程作为目标调用 unpark。
- 其他某个线程中断当前线程。
- 该调用不合逻辑地(即毫无理由地)返回。

#### parkNanos

此函数表示在许可可用前禁用当前线程，并**最多等待指定的等待时间**。具体函数如下。

```java
public static void parkNanos(Object blocker, long nanos) {
    if (nanos > 0) { // 时间大于0
        // 获取当前线程
        Thread t = Thread.currentThread();
        // 设置Blocker
        setBlocker(t, blocker);
        // 获取许可，并设置了时间
        UNSAFE.park(false, nanos);
        // 设置许可
        setBlocker(t, null);
    }
}
```

#### parkUntil

此函数表示在**指定的时限前**禁用当前线程，除非许可可用, 具体函数如下:

```java
public static void parkUntil(Object blocker, long deadline) {
    // 获取当前线程
    Thread t = Thread.currentThread();
    // 设置Blocker
    setBlocker(t, blocker);
    UNSAFE.park(true, deadline);
    // 设置Blocker为null
    setBlocker(t, null);
}
```

#### unpark

此函数表示

- 如果给定线程的许可**尚不可用，则使其可用**。
- 如果线程**在 park 上受阻塞，则它将解除其阻塞状态**。否则，**保证下一次调用 park 不会受阻塞**。
- 如果给定线程尚未启动，则无法保证此操作有任何效果。具体函数如下:

```java
public static void unpark(Thread thread) {
    if (thread != null) // 线程为不空
        UNSAFE.unpark(thread); // 释放该线程许可
}
```

## 深入理解

### Thread.sleep()和Object.wait()的区别

- Thread.sleep()**不会释放占有的锁**，Object.wait()**会释放占有的锁**；
- Thread.sleep()**必须传入时间**，Object.wait()**可传可不传**，不传表示一直阻塞下去；
- Thread.sleep()**到时间了会自动唤醒，然后继续执行**；
- Object.wait()不带时间的，**需要另一个线程使用Object.notify()唤醒**；
- Object.wait()带时间的，假如没有被notify，**到时间了会自动唤醒**，这时又分两种情况，**一是立即获取到了锁，线程自然会继续执行**；**二是没有立即获取锁，线程进入同步队列等待获取锁**；

其实，他们俩最大的区别就是**Thread.sleep()不会释放锁资源，Object.wait()会释放锁资源。**

### Object.wait()和Condition.await()的区别

- Object.wait()和Condition.await()的原理是基本一致的，**不同的是Condition.await()底层是调用LockSupport.park()来实现阻塞当前线程的**。

- 实际上，它在阻塞当前线程之前还干了两件事，一是**把当前线程添加到条件队列**中，二是**“完全”释放锁**，也就是让state状态变量变为0，然后才是调用LockSupport.park()阻塞当前线程。

### Thread.sleep()和LockSupport.park()的区别

从功能上来说，Thread.sleep()和LockSupport.park()方法类似，都是**阻塞当前线程的执行**，且都**不会释放当前线程占有的锁资源**；

- Thread.sleep()没法从外部唤醒，只能自己醒过来；

- LockSupport.park()方法可以被另一个线程调用LockSupport.unpark()方法唤醒；

- Thread.sleep()方法声明上抛出了InterruptedException中断异常，所以调用者需要捕获这个异常或者再抛出；

- LockSupport.park()方法不需要捕获中断异常；

- Thread.sleep()本身就是一个native方法；

- LockSupport.park()底层是调用的Unsafe的native方法；

### Object.wait()和LockSupport.park()的区别

二者都会阻塞当前线程的运行

- Object.wait()方法需要在synchronized块中执行（因为它会释放锁）；

- LockSupport.park()可以在任意地方执行（它不会释放锁）；

- Object.wait()方法声明抛出了中断异常，调用者需要捕获或者再抛出；

- LockSupport.park()不需要捕获中断异常；

- Object.wait()不带超时的，需要另一个线程执行notify()来唤醒，但不一定继续执行后续内容；

- LockSupport.park()不带超时的，需要另一个线程执行unpark()来唤醒，一定会继续执行后续内容；

park()/unpark()底层的原理是“二元信号量”，你可以把它相像成**只有一个许可证**的Semaphore，只不过这个信号量在重复执行unpark()的时候也不会再增加许可证，最多只有一个许可证。

### wait/notify和park/unpark的区别

- 必须先执行wait后执行notify，否则如果后执行wait，线程将一直阻塞；
- park和unpark可以以任意顺序执行，灵活度更高



# AQS

## 简介

- AQS是一个用来**构建锁和同步器的框架**。
- 使用AQS能简单且高效地构造出应用广泛的大量的同步器，比如我们提到的ReentrantLock，Semaphore，其他的诸如ReentrantReadWriteLock，SynchronousQueue，FutureTask等等皆是基于AQS的。
- 当然，我们自己也能利用AQS非常轻松容易地构造出符合我们自己需求的同步器。

## 核心思想

- 如果被请求的**共享资源空闲**，则将当前请求资源的**线程设置为有效的工作线程**，并且将**共享资源设置为锁定状态**。
- 如果被请求的**共享资源被占用**，那么就需要**一套线程阻塞等待以及被唤醒时锁分配的机制**，这个机制AQS是用**CLH队列锁**实现的，即**将暂时获取不到锁的线程加入到队列**中。

> CLH(Craig,Landin,and Hagersten)队列是一个虚拟的**双向队列**(虚拟的双向队列即不存在队列实例，仅存在结点之间的关联关系)。
>
> AQS是将每条请求共享资源的**线程**封装成**一个CLH锁队列的一个结点(Node)**来实现锁的分配。）

- AQS使用一个int成员变量来表示**同步状态**，通过内置的FIFO队列来完成**获取资源线程**的**排队**工作。AQS使用CAS对该**同步状态**进行原子操作实现对其值的修改。

```java
private volatile int state;//共享变量，使用volatile修饰保证线程可见性
```

- 状态信息通过protected类型的getState，setState，compareAndSetState进行操作

```java
//返回同步状态的当前值
protected final int getState() {  
        return state;
}
 // 设置同步状态的值
protected final void setState(int newState) { 
        state = newState;
}
//原子地(CAS操作)将同步状态值设置为给定值update如果当前同步状态的值等于expect(期望值)
protected final boolean compareAndSetState(int expect, int update) {
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
```

## 资源共享方式

AQS定义两种资源共享方式

- Exclusive(**独占**)：只有一个线程能执行，如ReentrantLock。又可分为公平锁和非公平锁：
  - 公平锁：按照线程在队列中的排队顺序，先到者先拿到锁
  - 非公平锁：当线程要获取锁时，无视队列顺序直接去抢锁，谁抢到就是谁的
- Share(**共享**)：多个线程可同时执行，如Semaphore/CountDownLatch。Semaphore、CountDownLatCh、 CyclicBarrier、ReadWriteLock等。

ReentrantReadWriteLock 可以看成是**组合式**，因为ReentrantReadWriteLock也就是读写锁**允许多个线程同时对某一资源进行读**。

不同的**自定义同步器**争用共享资源的方式也不同。自定义同步器在实现时**只需要实现共享资源 state 的获取与释放方式**即可，至于具体线程等待队列的维护(如获取资源失败入队/唤醒出队等)，AQS已经在上层已经帮我们实现好了。

## AQS数据结构

- `AbstractQueuedSynchronizer`类底层的数据结构是使用`CLH(Craig,Landin,and Hagersten)队列`是一个**虚拟的双向队列**(虚拟的双向队列即不存在队列实例，仅存在结点之间的关联关系)。
- AQS是将每条请求共享资源的**线程**封装成一个CLH锁队列的一个**结点**(Node)来实现锁的分配。
- 其中Sync queue，即同步队列，是**双向链表**，包括head结点和tail结点，head结点主要用作后续的调度。
- 而Condition queue不是必须的，其是一个单向链表，只有当使用Condition时，才会存在此单向链表。并且可能会有多个Condition queue。

![](https://blogpicture2022.oss-cn-hangzhou.aliyuncs.com/202207311913148.png)

## 总结

对于AbstractQueuedSynchronizer的分析，最核心的就是sync queue的分析。

- 每一个结点都是由前一个结点唤醒
- 当结点发现前驱结点是head并且尝试获取成功，则会轮到该线程运行。
- condition queue中的结点向sync queue中转移是通过signal操作完成的。
- 当结点的状态为SIGNAL时，表示后面的结点需要运行。

# ReentrantLock

## 源码分析

ReentrantLock**实现了Lock接口**，Lock接口中定义了lock与unlock相关操作，并且还存在newCondition方法，表示**生成一个条件**。

```java
public class ReentrantLock implements Lock, java.io.Serializable
```

### 内部类

ReentrantLock总共有三个内部类，并且三个内部类是紧密相关的，下面先看三个类的关系。

![](https://blogpicture2022.oss-cn-hangzhou.aliyuncs.com/202207311920689.png)

ReentrantLock类内部总共存在Sync、NonfairSync、FairSync三个类，NonfairSync与FairSync类继承自Sync类，Sync类继承自AbstractQueuedSynchronizer抽象类。
