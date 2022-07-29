> JUC 指的是`java.util.concurrent`、`java.util.concurrent.atomic`、`java.util.concurrent.locks`这三个包

#  CAS（比较并交换）

线程安全的实现方法包含:

- **互斥同步**: synchronized 和 ReentrantLock
- **非阻塞同步**: CAS, AtomicXXXX
- **无同步方案**: 栈封闭，Thread Local，可重入代码

## CAS概念

- CAS的全称为Compare-And-Swap，直译就是**对比交换**，是一条CPU的原子指令，其作用是让CPU**先进行比较两个值是否相等，然后原子地更新某个位置的值（CAS操作需要输入两个数值，一个旧值(期望操作前的值)和一个新值，在操作期间先比较下在旧值有没有发生变化，如果没有发生变化，才交换成新值，发生了变化则不交换。）。**
- 其实现方式是基于硬件平台的汇编指令，就是说CAS是**靠硬件实现**的，JVM**只是封装了汇编调用**，那些AtomicInteger类便是使用了这些封装后的接口。
- CAS操作是原子性的，所以多线程并发使用CAS更新数据时，可以不使用锁。JDK中大量使用了CAS来更新数据而**防止加锁**(synchronized 重量级锁)来保持原子更新。

## CAS存在的问题

### ABA问题

- 因为CAS需要在操作值的时候，检查值有没有发生变化，比如没有发生变化则更新，但是如果一个值**原来是A，变成了B，又变成了A**，那么使用CAS进行检查时则会发现它的**值没有发生变化**，但是**实际上却变化了**。
- ABA问题的解决思路就是**使用版本号**。在变量前面追加上版本号，每次变量更新的时候把版本号加1，那么A->B->A就会变成1A->2B->3A。
- 从Java 1.5开始，JDK的Atomic包里提供了一个类**AtomicStampedReference**来解决ABA问题。这个类的compareAndSet方法的作用是**首先检查当前引用是否等于预期引用**，并且**检查当前标志是否等于预期标志**，如果全部相等，则以原子方式将该引用和该标志的值设置为给定的更新值。

### 循环时间长开销大

- **自旋**CAS如果长时间不成功，会给CPU带来非常大的执行开销。
- 如果JVM能支持处理器提供的pause指令，那么效率会有一定的提升，pause指令有两个作用：
  - 它可以**延迟流水线执行命令**(de-pipeline)，使CPU不会消耗过多的执行资源，延迟的时间取决于具体实现的版本，在一些处理器上延迟时间是零
  - 它可以**避免在退出循环的时候**因**内存顺序冲突(**Memory Order Violation)而引起**CPU流水线被清空**(CPU Pipeline Flush)，从而提高CPU的执行效率。

### 只能保证一个共享变量的原子操作

- 对多个共享变量操作时，循环CAS就无法保证操作的原子性，这个时候就可以用**锁**。
- 从Java 1.5开始，JDK提供了**AtomicReference**类来保证引用对象之间的原子性，就可以把**多个变量放在一个对象里**来进行CAS操作。

# Unsafe类

- Unsafe是位于sun.misc包下的一个类，主要提供一些用于**执行低级别、不安全操作**的方法，Unsafe提供的API大致可分为**内存操作**、**CAS**、**Class相关**、**对象操作**、**线程调度**、**系统信息获取**、**内存屏障**、**数组操作**等几类
- 总体功能如图所示

![](https://blogpicture2022.oss-cn-hangzhou.aliyuncs.com/202206231807474.png)

## Unsafe与CAS

Unsafe只提供了**3种CAS方法**：compareAndSwapObject、compareAndSwapInt和compareAndSwapLong。都是**native**方法。

```java
public final native boolean compareAndSwapObject(Object paramObject1, long paramLong, Object paramObject2, Object paramObject3);

public final native boolean compareAndSwapInt(Object paramObject, long paramLong, int paramInt1, int paramInt2);

public final native boolean compareAndSwapLong(Object paramObject, long paramLong1, long paramLong2, long paramLong3);
```

## 其他功能

- Unsafe 提供了**硬件级别的操作**，比如说**获取某个属性在内存中的位置**，比如说**修改对象的字段值，即使它是私有的**。不过 Java 本身就是为了屏蔽底层的差异，对于一般的开发而言也很少会有这样的需求。

比方说：

```java
public native long staticFieldOffset(Field paramField); 
```

这个方法可以用来获取给定的 paramField 的**内存地址偏移量**，这个值**对于给定的 field 是唯一的且是固定不变**的。

再比如说：

```java
public native int arrayBaseOffset(Class paramClass);
public native int arrayIndexScale(Class paramClass);
```

前一个方法是用来**获取数组第一个元素的偏移地址**，后一个方法是用来**获取数组的转换因子**即数组中元素的**增量地址**的。

最后看三个方法：

```java
public native long allocateMemory(long paramLong);
public native long reallocateMemory(long paramLong1, long paramLong2);
public native void freeMemory(long paramLong);
```

分别用来**分配内存**，**扩充内存**和**释放内存**的。

# Atomic原子类

## 基本类型

- AtomicBoolean: 原子更新**布尔**类型。
- AtomicInteger: 原子更新**整型**。
- AtomicLong: 原子更新**长整型**。

以AtomicInteger为例：

### 常用API

```java
public final int get()：获取当前的值
public final int getAndSet(int newValue)：获取当前的值，并设置新的值
public final int getAndIncrement()：获取当前的值，并自增
public final int getAndDecrement()：获取当前的值，并自减
public final int getAndAdd(int delta)：获取当前的值，并加上预期的值
void lazySet(int newValue): 最终会设置成newValue,使用lazySet设置值后，可能导致其他线程在之后的一小段时间内还是可以读到旧的值。因为lasyset只在写前加入了storestore屏障没有在写后加storeload屏障
```

### 源码解析

```java
public class AtomicInteger extends Number implements java.io.Serializable {
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long valueOffset;
    static {
        try {
            //用于获取value字段相对当前对象的“起始地址”的偏移量
            valueOffset = unsafe.objectFieldOffset(AtomicInteger.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }

    private volatile int value;

    //返回当前值
    public final int get() {
        return value;
    }

    //递增加detla
    public final int getAndAdd(int delta) {
        //三个参数，1、当前的实例 2、value实例变量的偏移量 3、当前value要加上的数(value+delta)。
        return unsafe.getAndAddInt(this, valueOffset, delta);
    }

    //递增加1
    public final int incrementAndGet() {
        return unsafe.getAndAddInt(this, valueOffset, 1);
    }
...
}
```

AtomicInteger 底层用的是volatile的变量和CAS来进行更改数据的。

- volatile**保证线程的可见性**，多线程并发时，一个线程修改数据，可以保证其它线程立马看到修改后的值
- CAS **保证数据更新的原子性**。

## 数组类型

- AtomicIntegerArray: 原子更新整型数组里的元素。

- AtomicLongArray: 原子更新长整型数组里的元素。

- AtomicReferenceArray: 原子更新引用类型数组里的元素。

### 常用API

- get(int index)：获取索引为index的元素值。

- compareAndSet(int i,E expect,E update): 如果当前值等于预期值，则以原子方式将数组位置i的元素设置为update值。

## 引用类型

Atomic包提供了以下三个类：

- AtomicReference: 原子更新引用类型。
- AtomicStampedReference: 原子更新引用类型, 内部使用**Pair来存储元素值及其版本号**。
- AtomicMarkableReferce: 原子更新**带有标记位**的引用类型。

这三个类提供的方法都差不多，首先**构造一个引用对象**，然后**把引用对象set进Atomic类**，然后调用compareAndSet等一些方法去进行原子操作，原理都是**基于Unsafe**实现，但AtomicReferenceFieldUpdater略有不同，更新的字段必须用volatile修饰。

```java
import java.util.concurrent.atomic.AtomicReference;

public class AtomicReferenceTest {
    
    public static void main(String[] args){

        // 创建两个Person对象，它们的id分别是101和102。
        Person p1 = new Person(101);
        Person p2 = new Person(102);
        // 新建AtomicReference对象，初始化它的值为p1对象
        AtomicReference ar = new AtomicReference(p1);
        // 通过CAS设置ar。如果ar的值为p1的话，则将其设置为p2。
        ar.compareAndSet(p1, p2);

        Person p3 = (Person)ar.get();
        System.out.println("p3 is "+p3);
        System.out.println("p3.equals(p1)="+p3.equals(p1));
    }
}

class Person {
    volatile long id;
    public Person(long id) {
        this.id = id;
    }
    public String toString() {
        return "id:"+id;
    }
}
```

### AtomicStampedReference解决ABA问题

- AtomicStampedReference主要维护包含一个**对象引用**以及一个**可以自动更新的整数"stamp"**的**pair对象**来解决ABA问题。

```java
public class AtomicStampedReference<V> {
    private static class Pair<T> {
        final T reference;  //维护对象引用
        final int stamp;  //用于标志版本
        private Pair(T reference, int stamp) {
            this.reference = reference;
            this.stamp = stamp;
        }
        static <T> Pair<T> of(T reference, int stamp) {
            return new Pair<T>(reference, stamp);
        }
    }
    private volatile Pair<V> pair;
    ....
    
    /**
      * expectedReference ：更新之前的原始值
      * newReference : 将要更新的新值
      * expectedStamp : 期待更新的标志版本
      * newStamp : 将要更新的标志版本
      */
    public boolean compareAndSet(V   expectedReference,
                             V   newReference,
                             int expectedStamp,
                             int newStamp) {
        // 获取当前的(元素值，版本号)对
        Pair<V> current = pair;
        return
            // 引用没变
            expectedReference == current.reference &&
            // 版本号没变
            expectedStamp == current.stamp &&
            // 新引用等于旧引用
            ((newReference == current.reference &&
            // 新版本号等于旧版本号
            newStamp == current.stamp) ||
            // 构造新的Pair对象并CAS更新
            casPair(current, Pair.of(newReference, newStamp)));
    }

    private boolean casPair(Pair<V> cmp, Pair<V> val) {
        // 调用Unsafe的compareAndSwapObject()方法CAS更新pair的引用为新引用
        return UNSAFE.compareAndSwapObject(this, pairOffset, cmp, val);
    }
```

- 如果元素值和版本号都没有变化，并且和新的也相同，返回true；
- 如果元素值和版本号都没有变化，并且和新的不完全相同，就构造一个新的Pair对象并执行CAS更新pair。

可以看到，java中的实现跟我们上面讲的ABA的解决方法是一致的。

- 首先，使用版本号控制；
- 其次，不重复使用节点(Pair)的引用，每次都新建一个新的Pair来作为CAS比较的对象，而不是复用旧的；
- 最后，外部传入元素值及版本号，而不是节点(Pair)的引用。

### AtomicMakableRefernce解决ABA问题

它不是维护一个版本号，而是**维护一个Boolean类型的标记**，标记值有修改。

## 字段类型

Atomic包提供了四个类进行原子字段更新：

- AtomicIntegerFieldUpdater: 原子更新整型的字段的更新器。
- AtomicLongFieldUpdater: 原子更新长整型字段的更新器。
- AtomicStampedFieldUpdater: 原子更新带有版本号的引用类型。
- AtomicReferenceFieldUpdater: 更新的字段必须用volatile修饰。

这四个类的使用方式都差不多，是**基于反射**的原子**更新字段**的值。要想原子地更新字段类需要两步:

- 第一步，因为原子更新字段类都是抽象类，每次使用的时候必须使用静态方法newUpdater()创建一个更新器，并且需要设置想要更新的类和属性。
- 第二步，更新类的字段必须使用public volatile修饰。

```java
public class TestAtomicIntegerFieldUpdater {

    public static void main(String[] args){
        TestAtomicIntegerFieldUpdater tIA = new TestAtomicIntegerFieldUpdater();
        tIA.doIt();
    }

    public AtomicIntegerFieldUpdater<DataDemo> updater(String name){
        return AtomicIntegerFieldUpdater.newUpdater(DataDemo.class,name);

    }

    public void doIt(){
        DataDemo data = new DataDemo();
        System.out.println("publicVar = "+updater("publicVar").getAndAdd(data, 2));
        /*
            * 由于在DataDemo类中属性value2/value3,在TestAtomicIntegerFieldUpdater中不能访问
            * */
        //System.out.println("protectedVar = "+updater("protectedVar").getAndAdd(data,2));
        //System.out.println("privateVar = "+updater("privateVar").getAndAdd(data,2));

        //System.out.println("staticVar = "+updater("staticVar").getAndIncrement(data));//报java.lang.IllegalArgumentException
        /*
            * 下面报异常：must be integer
            * */
        //System.out.println("integerVar = "+updater("integerVar").getAndIncrement(data));
        //System.out.println("longVar = "+updater("longVar").getAndIncrement(data));
    }

}

class DataDemo{
    public volatile int publicVar=3;
    protected volatile int protectedVar=4;
    private volatile  int privateVar=5;

    public volatile static int staticVar = 10;
    //public  final int finalVar = 11;

    public volatile Integer integerVar = 19;
    public volatile Long longVar = 18L;

}
```

对于AtomicIntegerFieldUpdater 的使用稍微有一些限制和约束，约束如下：

- **字段必须是volatile类型的**，在线程之间共享变量时保证立即可见
- **字段的描述类型**(修饰符public/protected/default/private)是与**调用者与操作对象字段的关系**一致。也就是说调用者能够直接操作对象字段，那么就可以反射进行原子操作。但是对于父类的字段，子类是不能直接操作的，尽管子类可以访问父类的字段。
- 只能是实例变量，**不能是类变量**，也就是说不能加static关键字。
- 只能是可修改变量，**不能是final变量**，因为final的语义就是不可修改。实际上final的语义和volatile是有冲突的，这两个关键字不能同时存在。
- 对于AtomicIntegerFieldUpdater和AtomicLongFieldUpdater只能修改int/long类型的字段，不能修改其包装类型(Integer/Long)。如果要修改包装类型就需要使用AtomicReferenceFieldUpdater。