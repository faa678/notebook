



java内存区域结构: 

![avatar](/home/faa/pictures/论文/内存结构.png)

线程共享：堆、方法区

线程私有：虚拟机栈、本地方法栈、程序计数器

1. 

程序计数器为什么是私有的：

​	字节码解释器通过pc读取指令，pc用于记录当前线程执行的位置和下一条指令的位置（如果执行native，undefined），当线程切换回来时能够知道上次执行到哪

虚拟机栈和本地方法栈私有：保证线程中的局部变量不被别的线程引用

​	虚拟机栈存储局部变量、操作数栈、常量池引用等。

​	本地方法栈存储本地方法服务。

2. 堆和方法区

   都存放共享资源。

   堆是最大内存，存放<font color=red>对象</font>

   方法区：类模板信息

   

3. 多线程好处

   高并发系统的基础

4. 上下文切换

   时间片轮转。当一个线程的时间片用完的时候就会重新处于就绪状态让给其他线程使用，当前任务在执行完cpu时间片切换到另一个任务前保存自己的状态，以便下次切换回来架子啊此任务状态。<font color=red>任务从保存到在加载的过程就是一次上下文切换</font>

5. 避免线程死锁
   * 破坏互斥条件
   * 破坏请求与保持条件（请求资源阻塞时，对已获得的资源保持不放）：一次性申请所有的资源
   * 破坏不剥夺条件（已获得资源未使用完前不能被其他线程强行剥夺）：申请不到资源主动释放占用的资源
   * 破坏循环等待条件：按序申请

6. sleep()和wait()
   * sleep()没有释放锁，wait()释放了锁
   * 都可以暂停线程
   * wait用于交互，sleep用于暂停
   * wait靠唤醒，wait(long timeout)自动，sleep自动

7. 为什么不能直接调用run()

   new 一个Thread，线程进入新建状态，调用start方法，底层调用native方法start0，启动一个新线程并进入就绪状态，分到时间片就可以运行了。自动回调run。

   直接调用run，会把run当成作为thread的一个普通方法在main线程下执行，没有启动新线程的行为。

8. synchronized（底层jvm C++ 实现）[详情参考](http://47.102.117.181:8080/archives/synchronized)
   * 修饰实例方法：相当于给当前对象加锁，进入同步代码前需要获得当前对象实例的锁
   * 修饰静态方法：给当前类加锁，Class模板，会作用与对于类的所有对象实例
     * <font color=yellow>如果一个线程A调用一个实例对象o(并非指锁)的非静态synchronized方法，线程B调用o所属类的静态synchronized方法，是允许的，<font color=red>不会发生互斥现象</font>。因为一个是对象锁，一个是类锁。</font>
   * 修饰代码块：对给定对象加锁，进入同步代码块前要获得给定对象的锁。
   
9. 单例模式（双重校验锁实现单例模式（线程安全））//确保类只有一个实例

```java
public class Singleton {
    

	private volatile static Singleton uniqueInstance;
    private Singleton() {
        
    }
    
    public static Singleton getUniqueInstance() {
        //判断对象是否已经实例化，如果没有，才继续执行
        if(uniqueInstance == null) {
            synchronized (Singleton.class) {
                if(uniqueInstance == null) {
                    /*--->查看初始化
                        uniqueInstance = new Singleton()分为三步：所以需要使用volatile修饰，禁止指令重排
                        为uniqueInstance分配内存空间
                        初始化uniqueInstance
                        将uniqueInstance指向分配的内存空间
                    */
                    uniqueInstance = new Singleton();
                }
            }
        }
        return uniqueInstance;
    }
    
}
```

synchronized的优化：（都是在jvm层面的优化）

​	偏向锁	轻量级锁	自旋锁	适应性自旋锁	锁消除	锁粗化

synchronized和ReentrantLock

* 两者都是可重入锁
* synchronized依赖于JVM，ReentrantLock依赖于API（jdk层面实现）
* ReentrantLock高级功能：
  * 等待可中断：通过lock.lockInterruptibly()实现中断，放弃等待
  * 可实现公平锁：可以指定是公平锁还是非公平锁，而synchronized只能是非公平；ReentrantLock默认非公平，可以通过构造方法ReentrantLock(boolean fair)指定是否公平
  * synchronized与wait()和notify()/notifyAll()结合可以实现等待/通知机制；ReentrantLock借助Condition接口的newCondition方法实现，对应await()，signal()，signalAll()。<font color=red>ReentrantLock结合Condition可以实现选择性通知，Condition实例的signalAll()方法只会唤醒注册在该Condition实例中的所有等待线程。</font>

10. volatile

    1. java内存模型（JMM）

       线程之间的共享变量存储在主存中，每个线程都有一个私有的本地内存（抽象概念），本地内存存储共享变量的副本，<font color=red>这就可能造成一个线程在本地内存中修改了变量副本后并写回主存后，另一个线程还在使用其本地内存的副本，造成数据不一致</font>。

       <img src="/home/faa/pictures/java内存模型.png" alt="java内存模型" style="zoom:50%;" />

       其中，线程之间的通信（必须经过主存）：

       * 线程 A 把本地内存中更新过的共享变量刷新到主存

       * 线程 B 到主存中读取

解决上述问题，使用volatile，保证共享变量的可见性，并且防止指令重排序。[详情参考](/home/faa/files/notebook/java/juc/JUC.md)

11. volatile和synchronized

    * volatile是synchronized的轻量级实现。

    * 性能上：volatile > synchronized

    * volatile只能修饰变量；synchronized可以修饰代码块和方法
    * 访问volatile变量不会阻塞，synchronized关键字可能会阻塞
    * volatile保证数据可见性，不能保证原子性和互斥性；synchronized都能保证
    * volatile用于解决变量在多个线程间可见性，synchronized解决多个线程访问资源同步性。

12. ThreadLocal   [详情参考](./ThreadLocal.md)




13. 线程池  [详情参考](./线程池.md)

    好处：

    	* 降低资源消耗，重复利用已创建的线程降低线程创建和销毁时的消耗
    	* 提高响应速度，任务到达时无需等待线程创建
    	* 提高线程可管理性
    实现 Runnable 和 Callable 区别

    * Runnable 不会返回结果或抛出异常，Callable 可以

    * 实现Callable接口创建线程，需要重写 call 函数，并使用FutureTask接收 call 的返回值，并且 Callable接口带泛型

    执行execute()方法和submit()区别

    	* execute 方法用于提交不需要返回值的任务，无法判断任务是否被线程池执行成功
    	* submit提交有返回值的任务。返回Future对象，可以判断是否执行成功，通过get方法获取返回值，get方法会阻塞当前线程直到完成，get(long timeout, TimeUnit unit)会阻塞当前线程一段时间后立即返回，此时当前任务还没有完成。

14. Atomic 原子类

    4类：

     	1. 基本类型
         * AtomicInteger
         * AtomicLong
         * AtomicBoolean
     	2. 数组类型
         * AtomicIntegerArray
         * AtomicLongArray
         * AtomicReferenceArray
     	3. 引用类型
         	* AtomicReference：引用类型原子类
         	* AtomicStampedReference：原子更新带有版本号的引用类型。将整数值与引用关联起来，可用于解决原子的更新数据和数据的版本号，可以解决使用CAS进行原子更新时可能出现的ABA问题
         	* AtomicMarkableReference：原子更新带有标记位的引用类型
     	4. 对象属性修改类型
         * AtomicIntegerFieldUpdater：原子更新整型字段的更新器
         * AtomicLongFiledUpdater：原子更新长整形字段的更新器



​	AtomicInteger类常用方法

```java
public final int get()//获取当前值
public final int getAndSet(int newValue)//获取当前值，并设置新值
public final int getAndIncrement()//获取当前值，并自增
public final int getAndDecrement()//获取当前值，并自减
public final int getAndAdd(int delta)//获取当前值，并加上预期值
boolean compareAndSet(int expect, int update)//如果当前输入的值等于预期值，则以原子方式将该值设置为输入值
public final void lazySet(int newValue)//最终设置为newValue，使用lazySet设置之后可能导致其他线程在之后的一小段时间内还是可以读到旧的值
```

AtomicInteger部分源码：

```java
 private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long valueOffset;

    static {
        try {
            valueOffset = unsafe.objectFieldOffset
                (AtomicInteger.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }

    private volatile int value;
```



AtomicInteger原理：

​	利用CAS(compare and swap) + volatile 和 native 方法保证原子操作，从而避免synchronized的高开销。

​	CAS的原理是拿期望的值和原本的值作比较，如果相同则更新成新的值。Unsafe类的objectFieldOffset方法是一个本地方法，用来拿到"原来的值"（value）的内存地址，返回值是valueOffset。

15. ReentrantLock（AQS）加锁和释放锁原理：

    * 基于AQS实现。

    * AQS对象内部有一个int类型的核心变量state，代表了加锁的状态，初始状态下，这个state值是0。

    * AQS对象内部还有一个关键变量，用来记录当前加锁的是哪个线程，初始状态是null

    线程1来调用ReentrantLock的lock方法尝试进行加锁，就是用CAS操作将state值从0变为1。如果之前没有加过锁，则state为0，线程1就可以加锁成功。

    ![avatar](/home/faa/pictures/AQS1.png)

    ReentrantLock只是一个外层api，内核中的机制实现都是依赖AQS组件的。

    锁互斥的实现：

    此时线程2来尝试加锁，发现state的值是1，说明已经被加锁，检查加锁线程变量，发现是线程1加的锁，所以线程2加锁失败。

    ![avatar](/home/faa/pictures/AQS2.png)

    然后，线程2会将自己放入AQS中的一个等待队列，等待线程1释放锁。

    ![avatar](/home/faa/pictures/AQS3.png)

    线程1释放锁：

    ![avatar](/home/faa/pictures/AQS4.png)

    然后从等待队列的队头获取线程2，线程2重新尝试加锁。

    ![avatar](/home/faa/pictures/AQS5.png)

16. AQS（AbstractQueuedSynchronizer）

原理：如果被请求的共享资源空闲，则将当前请求资源的线程设置为有效，并置为锁定。如果共享资源被占用，就需要一套线程阻塞等待以及被唤醒时锁分配的机制，这个机制使用CLH队列（虚拟的双向队列，不存在队列实例，仅存在结点之间的关联关系）锁实现的，即将暂时获取不到的锁的线程加入到队列中。

AQS原理：

​	![avatar](/home/faa/pictures/AQS.png)

AQS使用int类型成员变量state表示同步状态，使用CAS对其进行原子操作实现对值的修改。

```java
private volatile int state; //共享变量，使用volatile修饰保证线程可见性
```

状态信息通过protected类型的getState，setState，compareAndSetState进行操作

```java
protected final int getState() {return state;}
protected final void setState(int newState) {
	state = newState;
}
protected final boolean compareAndSetState(int expect, int update) {
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
```

AQS对资源的共享方式：

Exclusive：只有一个线程能执行，如ReetrantLock。又可分为公平锁和非公平锁：

	* 公平锁：按照线程在队列中的排队顺序，先到者先拿到锁
	* 非公平锁：当线程要获取锁时，无视队列顺序直接去抢锁，谁抢到就是谁的

Share：多个线程可同时执行，如Semaphore、CountDownLatch

​	Semaphore、CountDownLatch、CyclicBarrier、ReadWriteLock等