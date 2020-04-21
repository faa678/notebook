



#### volatile关键字与内存可见性：

实现原则：

通过内存屏障和禁止重排序优化来实现内存可见性：

	* 对共享内存进行写操作后，加入一条 store 屏障指令，强制将共享变量的值刷新到主存中，并使其他工作内存中的副本无效
	* 对共享变量进行读操作前，加入一条load屏障指令，强制从主存中将最新值刷新到工作内存



1. 内存可见性：

代码示例：

```java
package com.faa.juc;

public class TestVolatile {

    public static void main(String[] args) {
        ThreadDemo td = new ThreadDemo();
        //ThreadDemo子线程
        new Thread(td).start();

        //main线程
        while(true) {
            if(td.getFlag()) {
                System.out.println("主线程读取到的flag值：" + td.getFlag());
                break;
            }
        }
    }
}

class ThreadDemo implements Runnable {

    private boolean flag = false;

    public boolean getFlag() {
        return flag;
    }

    public void setFlag(boolean flag) {
        this.flag = flag;
    }

    //重写 run 方法
    @Override
    public void run() {
        try {
            Thread.sleep(200);//使得未来得及将ThreadDemo子线程中的 flag = true 同步到主存中时，main线程就已经进来了
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        flag = true;
        System.out.println("After changing flag = " + getFlag());

    }
}
```

执行结果：

​	产生<font color=red>死循环</font>，main线程获取到的 flag 值还是 false

```
After changing flag = true
```

解析：

​	产生死循环的原因是，<font color=red>内存可见性问题</font>：当多个线程操作共享数据时，彼此不可见。

![内存可见性](https://upload-images.jianshu.io/upload_images/11531502-920d52c60aec3dd3.png?imageMogr2/auto-orient/strip|imageView2/2/w/585/format/webp)

解决方法：

​	使用 synchronized互斥锁 加锁，使while循环重复到主存中读取flag。

```java
while(true) {
    synchronized (td) {
        if (td.isFlag()) {
            System.out.println("主线程读取到的flag值：" + td.isFlag());
            break;
        }
    }
}
```

缺点：效率低。加锁后，每次只能有一个线程访问，当一个线程持有锁时，其他的线程就会<font color=red>阻塞</font>。不想加锁，又要解决内存可见性问题，就可以使用 <font color=red>volatile</font> 关键字。



volatile: 当多个线程操作共享数据时，可以<font color=blue>保证内存中的数据可见</font>，相较于 synchronized 是一种较为<font color=blue>轻量级</font>的同步策略。使用 volatile 修饰共享数据，会<font color=blue>及时地把线程缓存中的数据刷新到主存中去，并使其他线程的副本无效，也可以理解为直接操作主存中的数据</font>。

```java
private volatile boolean flag = false;
```

注意：

1. volatile 不具备 "互斥性"（互斥性：当一个线程持有锁时，其他线程进不来）；

 	2. volatile 不能保证变量的 "原子性"（原子性：操作是否可拆分）。



#### 原子性

##### 1. 理解原子性

i++ 的原子性问题：

```java
int i = 10;
int j = i++;  //j == 10
```

i++ 不是原子操作，它的操作实际上分为3个步骤：读-改-写

```java
int tmp = i;
i = i + 1;
int j = tmp;
```

代码示例：

```java
package com.faa.juc;

/*
i++问题
 */

public class TestAtomic {
    public static void main(String[] args) {
        AtomicDemo ad = new AtomicDemo();
        for(int i = 0; i < 10; i++) {
            new Thread(ad).start();
        }
    }
}

class AtomicDemo implements Runnable {

    private int serialNumber = 0;//线程共享

    public int getSerialNumber() {
        return serialNumber++;
    }

    @Override
    public void run() {
        try {
            Thread.sleep(200);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(Thread.currentThread().getName() + ": " + getSerialNumber());
    }
}
```

执行结果：

可能会出现以下结果：

```
Thread-1: 0	//和 Thread-0 结果一样
Thread-5: 4
Thread-6: 5
Thread-4: 3
Thread-3: 2
Thread-2: 1
Thread-7: 6
Thread-0: 0	//和 Thread-1 结果一样
Thread-9: 8
Thread-8: 7
```

解析：

出现了 i++ 的原子性问题。如图解：

![avatar](https://upload-images.jianshu.io/upload_images/11531502-9f3f6172bf3b45fc.png?imageMogr2/auto-orient/strip|imageView2/2/w/577/format/webp)

即使此时使用 volatile 修饰共享变量：

```java
private volatile int serialNumber = 0;
```

执行结果仍然可能出现线程安全问题。因为 volatile 不能保证原子性，所以，使用 volatile 修饰 serialNumber 时，当两个线程同时获得了内存中的 0 ，又同时自增，同时写入内存，结果还是可能出现重复数据，因为使用 volatile 只是相当于所有线程都是在主存中操作数据，但是<font color=red>不具备互斥性</font>。



##### 2. 原子变量

jdk1.5 后 <font color=purple>`java.util.concurrent.atomic`</font> 包下提供了常用的原子变量：

 * <font color=blue>volatile</font> 保证内存可见性

 * <font color=blue>CAS（Compare-And-Swap）</font> 算法保证数据的原子性

   * CAS 算法是硬件对于并发操作共享数据的支持

   * CAS 包含了3个操作数：

     > 内存值 			V
     >
     > 旧的预期值    A
     >
     > 更新值 			B

     当且仅当 V == A 时，将 B 赋给 V，即 V = B，否则不做任何操作。

对于上面的 i++ 问题来说，假如线程 1 和 线程 2 获取 V == 0，线程 1 进行比较和替换操作：V == A == 0 && V = (B == 1)，此时线程 2 进行比较和替换操作：(V == 0) != (A == 1)，此时什么都不做，也就保证了 i++ 的原子性。

因此，可以如下使用原子变量改进 i++ 问题：

```java
    private AtomicInteger serialNumber = new AtomicInteger();

    public int getSerialNumber() {
        return serialNumber.getAndIncrement();
    }
```



但是会不会同时有两个线程同时比较且同时替换？

​	<font color=red>不会</font>：

1. 原语由若干条指令组成，用于完成一定功能的一个过程；

 	2. 即原语必须是连续的，在执行过程中不允许被中断。



模拟CAS算法：

```java
package com.faa.juc;

public class SimulateCAS {
    public static void main(String[] args) {
        final CompareAndSwap cas = new CompareAndSwap();

        for (int i = 0; i < 10; i++) {
            new Thread(new Runnable() {
                @Override
                public void run() {
                    int expectedValue = cas.get();
                    boolean b = cas.compareAndSwap(expectedValue, (int) (Math.random() * 101));
                    System.out.println(b);
                }
            }).start();
        }
    }
}

class CompareAndSwap {

    private int value;

    //获取内存值
    public synchronized int get() {
        return value;
    }

    //比较并交换
    public synchronized boolean compareAndSwap(int expectedValue, int newValue) {

        int oldValue = value;

        if(expectedValue == value) {
            this.value = newValue;
        }
        return oldValue == expectedValue;
    }
}
```



#### ConcurrentHashMap <font color=blue>锁分段机制</font>（jdk1.7）

1. ConcurrentHashMap 是一个线程安全的 hash 表。HashMap 是线程不安全的，HashTable 是线程安全的。但是 HashTable 是将整个 hash 表锁起来，当有多个线程访问时，同一时间只能有一个线程获得锁，并行变成串行效率低。jdk1.7提供了 ConcurrentHashMap，采用了锁分段机制。

![avatar](https://upload-images.jianshu.io/upload_images/11531502-046e702837bca8cd.png?imageMogr2/auto-orient/strip|imageView2/2/w/902/format/webp)



ConcurrentHashMap 默认分成了16个 Segment，每个 Segment 对应一个 Hash 表，且都有独立的锁，实现了并行访问。但是，jdk1.8 不再使用锁分段机制，而是采用了 synchronized + CAS 实现多线程安全。



#### CountDownLatch闭锁

<font color=purple>`CountDownLatch`</font>是一个同步辅助类，在完成一组正在其他线程中执行的操作之前，允许一个或多个线程一直等待。

如下，要实现使用 10 个线程分别执行任务，并想要在执行完成后计算出总共消耗的时间：

未加闭锁前代码：

```java
public class TestCountDownLatch {

    public static void main(String[] args) {

        LatchDemo ld = new LatchDemo();
        long start = System.currentTimeMillis();

        for(int i = 0; i < 3; i++) {
            new Thread(ld).start();
        }

        long end = System.currentTimeMillis();
        System.out.println("耗费时间：" + (end - start));

    }
}

class LatchDemo implements Runnable {

    @Override
    public void run() {

        for (int i = 0; i < 3; i++) {
            if (i % 2 == 0) {
                System.out.println(i);
            }
        }

    }
}
```

执行结果：

```
0
2
0
2
耗费时间：2
0
2
```

解析：

在子线程执行结束前main线程就执行了。

加上闭锁操作后：

```java
public class TestCountDownLatch {

    public static void main(String[] args) {
        final CountDownLatch latch = new CountDownLatch(3);//有几个线程访问这个参数就是几

        LatchDemo ld = new LatchDemo(latch);
        long start = System.currentTimeMillis();

        for(int i = 0; i < 3; i++) {
            new Thread(ld).start();
        }

        try {
            latch.await();//子线程执行完前先等待
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        long end = System.currentTimeMillis();
        System.out.println("耗费时间：" + (end - start));

    }
}

class LatchDemo implements Runnable {

    private CountDownLatch latch;

    public LatchDemo(CountDownLatch latch) {
        this.latch = latch;
    }

    @Override
    public void run() {
        synchronized (this) {
            try {
                for (int i = 0; i < 3; i++) {
                    if (i % 2 == 0) {
                        System.out.println(i);
                    }
                }
            } finally{
                latch.countDown();//每执行完一个线程就递减一个
            }
        }
    }
}
```

执行结果：

```
0
2
0
2
0
2
耗费时间：3
```

解析：

<font color=purple>`CountDownLatch`</font>: 闭锁，在完成某些运算时，只有其他所有线程的运算全部完成，当前运算才继续执行。主要使用 <font color=purple>`latch.countDown()`</font> 和 <font color=purple>`latch.await()`</font> 实现



#### 线程创建方式

1. 继承 Thread 类，复写 run()，Thread 类就是实现的 Runnable 接口

   * 使用时通过调用Thread的 start()（该方法调用的是 native 方法 start0()），再调用创建线程的 run()，不同线程的 run 方法里的代码交替执行。

   * 其中：

     * run()：用于封装线程运行的任务代码，直接用创建的线程对象调用，并没有产生新的线程，而就像一个普通的方法调用，在<font color=red>当前线程的上下文</font>中执行

     * start()：共有两个作用

       > <font color=red>开启新线程</font>，确保在新的线程上下文中运行
       >
       > 新线程执行 run()

```java
public class TestThread {

    public static void main(String[] args) {
        for(int i = 0; i < 3; i++) {
            new ThreadAction().start();
        }
    }
}
class ThreadAction extends Thread {
    @Override
    public void run() {
        for(int i = 0; i < 3; i++) {
            System.out.println(Thread.currentThread().getName() + ": " + i);//i是线程私有的
        }
    }
}
```



2. 实现 Runnable 接口

```java
public class TestThread {

    private static ThreadAction ta = new ThreadAction();

    public static void main(String[] args) {
        for(int i = 0; i < 3; i++) {
            new Thread(ta).start();
        }
    }
}
class ThreadAction implements Runnable {
    @Override
    public void run() {
        for(int i = 0; i < 3; i++) {
            System.out.println(Thread.currentThread().getName() + ": " + i);//i是线程私有的
        }
    }
}
```



3. 实现 Callable 接口

```java
public class TestThread {

    private static ThreadAction ta = new ThreadAction();

    public static void main(String[] args) {
        FutureTask<Integer> ft = new FutureTask<Integer>(ta);
//        for(int i = 0; i < 3; i++) {//为什么加了循环也只能有一个线程
            new Thread(ft).start();
            try {
                System.out.println("结果：" + ft.get());//当上面的子线程执行完后，才会打印结果。跟闭锁一样，所以FutureTask也可以用于闭锁
            } catch (InterruptedException e) {
                e.printStackTrace();
            } catch (ExecutionException e) {
                e.printStackTrace();
            }
//        }
    }
}
class ThreadAction implements Callable<Integer> {
    @Override
    public Integer call() throws Exception {
        for(int i = 0; i < 3; i++) {
            System.out.println(i);
        }
        return 1;
    }
}
```

Callable接口和实现Runable接口的区别就是，Callable带泛型，其call方法有返回值。使用的时候，需要用FutureTask来接收返回值。而且它也要等到线程执行完调用get方法才会执行，也可以用于闭锁操作。



4. 线程池

   * 提供了一个线程队列，队列中保存着所有等待状态的线程。避免了创建与销毁的额外开销，提高了响应速度

   * 线程池体系：

     java.util.concurrent.Executor：根接口

     ​		|--ExecutorService：子接口

     ​				|--ThreadPoolExecutor：线程池的实现类

     ​				|--ScheduledExecutorService子接口：负责线程调度

     ​						|--ScheduledThreadPoolExecutor：继承ThreadPoolExecutor，实现ScheduledExecutorService

   * 工具类：Executors    （<font color=red>阿里巴巴不推荐</font>）

     * ExecutorService newFixedThreadPool()：创建固定大小的线程池
     * ExecutorService newCachedThreadPool()：缓存线程池，线程池内线程数量不固定，可以根据需求自动更改
     * ExecutorService newSingleThreadExecutor()：创建只有一个线程的线程池，线程池中只有一个线程
     * ScheduledExecutorService newScheduledThreadPool()：创建固定大小的线程池，可以延迟或定时地执行任务

```java
public class TestThreadPool {

    public static void main(String[] args) throws ExecutionException, InterruptedException {

        ThreadPoolDemo td = new ThreadPoolDemo();
        //创建线程
        ExecutorService pool = Executors.newFixedThreadPool(5);
        //为线程池中线程分配任务
        //Runnable
        for(int i = 0; i < 10; i++) {
            pool.submit(td);
        }

        //Callable
        for(int k = 0; k < 10; k++) {
            Future<Integer> future = pool.submit(new Callable<Integer>() {
                @Override
                public Integer call() throws Exception {
                    int i = 0;
                    for (int j = 0; j < 10; j++)
                        i++;
                    return i;
                }
            });
            System.out.println(future.get());
        }
        //关闭线程池
        pool.shutdown();

        //可以延迟执行线池
        ScheduledExecutorService spool = Executors.newScheduledThreadPool(5);
        for(int i = 0; i < 5; i++) {
            Future<Integer> future = spool.schedule(new Callable<Integer>() {
                @Override
                public Integer call() throws Exception {
                    int num = new Random().nextInt(100);
                    System.out.println(Thread.currentThread().getName() + num);
                    return num;
                }
            }, 3, TimeUnit.SECONDS);
            System.out.println(future.get());//get方法有闭锁功能，单纯输出常亮内容如5的话不能体现延迟
        }
        spool.shutdown();
    }

}
class ThreadPoolDemo implements Runnable {

    private int i = 0;

    @Override
    public void run() {
        while(i <= 10) {
            System.out.println(Thread.currentThread().getName() + ": " + i++);
        }
    }
}
```



#### 同步锁

用于解决多线程安全问题的方式：

synchronized（隐式锁）：

1. 同步代码块
2. 同步方法

jdk1.5 后：

​	同步锁 Lock（显示锁），通过 <font color=purple>`lock()`</font> 方法上锁，必须通过 <font color=purple>`unlock()`</font>（为了保证锁能被释放，放在 finally 中）方法释放锁

卖票问题：

加同步锁前：

```java
public class TestLock {

    public static void main(String[] args) {
        Ticket ticket = new Ticket();
        new Thread(ticket, "1号窗口").start();
        new Thread(ticket, "2号窗口").start();
        new Thread(ticket, "3号窗口").start();
    }
}

class Ticket implements Runnable {

    private int tickets = 100;

    @Override
    public void run() {
        while(true) {
            if (tickets > 0) {
                try {
                    Thread.sleep(200);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(Thread.currentThread().getName() + "完成售票，余票为：" + --tickets);
            }
        }
    }
}
```

多个线程同时操作共享数据tickets，所以会出现线程安全问题。会出现一张票卖了好几次或者票数为负数的情况。可以用同步代码块和同步方法解决，也可以用同步锁解决。

加同步锁后：

```java
public class TestLock {

    public static void main(String[] args) {
        Ticket ticket = new Ticket();
        new Thread(ticket, "1号窗口").start();
        new Thread(ticket, "2号窗口").start();
        new Thread(ticket, "3号窗口").start();

    }

}

class Ticket implements Runnable {

    private int tickets = 100;

    private Lock lock = new ReentrantLock();

    @Override
    public void run() {
        while(true) {
            lock.lock();
            try {
                if (tickets > 0) {
                    try {
                        Thread.sleep(200);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    System.out.println(Thread.currentThread().getName() + "完成售票，余票为：" + --tickets);
                }
            } finally {
                lock.unlock();
            }
        }
    }
}
```

使用 Lock 对象的 <font color=purple>`lock()`</font> 方法上锁，在 finally 中用 <font color=purple>`unlock()`</font> 方法上锁。



#### 生产者消费者：

1. synchronized实现

生产者消费者模式是等待唤醒机制的一个经典案例：

```java
package com.faa.juc;

public class TestProducerAndConsumer {

    public static void main(String[] args) {
        Commodities com = new Commodities();

        Producer p = new Producer(com);
        Consumer c = new Consumer(com);

        new Thread(p, "生产者").start();
        new Thread(c, "消费者").start();

    }

}

class Commodities {
    private int commodities = 0;
    public synchronized void sellCommdities() {
        if(commodities <= 0) {
            System.out.println("商品空");
            try {
                this.wait();//商品为空的时候就等待其他线程将其唤醒
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        } else {
            System.out.println(Thread.currentThread().getName() + ": " + --commodities);
            this.notifyAll();
        }
    }

    public synchronized void addCommdities() {
        if(commodities >= 10) {
            System.out.println("商品满");
            try {
                this.wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        } else {
            System.out.println(Thread.currentThread().getName() + ": " + ++commodities);
            this.notifyAll();
        }
    }
}

class Producer implements Runnable {
    private Commodities com;
    public Producer(Commodities com) {
        this.com = com;
    }
    @Override
    public void run() {
        for(int i = 0; i < 20; i++) {
            com.addCommdities();
        }
    }
}

class Consumer implements Runnable {
    private Commodities com;
    public Consumer(Commodities com) {
        this.com = com;
    }
    @Override
    public void run() {
        for(int i = 0; i < 20; i++) {
            com.sellCommdities();
        }
    }
}
```

* 这里使用等待唤醒机制，this.wait()释放当前锁资源，等待唤醒；this.notifyAll()唤醒。与synchronized绑定。<font color=red>synchronized修饰非静态方法，是对this加锁；修饰静态方法，是对Class对象加锁</font> --> 线程八锁问题。

* 但是此处有可能会发生程序不能停止。出现这种情况的原因是有一个线程在等待，但是另一个线程没有执行机会了，唤醒不了这个等待的线程了；解决办法是将addCommodities()和sellCommodities()中的else去掉，无论商品是否空或者满，都执行唤醒操作。
* 但是，此时如果设置两个生产者线程，两个消费者线程进行资源抢占，会出现负数的情况：
  * 一个消费者抢到执行权，发现commodities为0，就等待，此时另一个消费者又抢到了执行权，commodities为0，还是等待，此时两个消费者线程在同一处等待唤醒。当生产者生产了一个commodity后，就会唤醒两个消费者，发现commodities == 1，同时消费，结果就出现了0 和 -1。这就是虚假唤醒。解决方法为将等待唤醒放在循环中，每次都去判断一下，如下：

```java
    public synchronized void sellCommdities() {
        while(commodities <= 0) {
            System.out.println("商品空");
            try {
                this.wait();//商品为空的时候就等待其他线程将其唤醒
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
            System.out.println(Thread.currentThread().getName() + ": " + --commodities);
            this.notifyAll();

    }

    public synchronized void addCommdities() {
        while(commodities >= 1) {//为了避免虚假唤醒，应该使用在循环中
            System.out.println("商品满");
            try {
                this.wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
            System.out.println(Thread.currentThread().getName() + ": " + ++commodities);
            this.notifyAll();

    }
```

2. 使用Lock锁实现等待唤醒

```java
class Commodities {
    private int commodities = 0;
    Lock lock = new ReentrantLock();
    private Condition condition = lock.newCondition();
    public void sellCommdities() {
        lock.lock();
        try {
            while (commodities <= 0) {
                System.out.println(Thread.currentThread().getName() + ": " + "商品空");
                try {
                    condition.await();//商品为空的时候就等待其他线程将其唤醒
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            System.out.println(Thread.currentThread().getName() + ": " + --commodities);
            condition.signalAll();
        } finally {
            lock.unlock();
        }

    }

    public void addCommdities() {
        lock.lock();
        try {
            while (commodities >= 1) {
                System.out.println(Thread.currentThread().getName() + ": " + "商品满");
                try {
                    condition.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            System.out.println(Thread.currentThread().getName() + ": " + ++commodities);
            condition.signalAll();
        } finally {
            lock.unlock();
        }

    }
}
```

使用Lock同步锁，就不需要synchronized关键字了，需要创建Lock实例和Condition实例，Condition对象的await()方法、signal()方法和signalAll()方法分别与wait()方法、notify()方法和notifyAll()方法对应。

3. 线程按序交替

题目介绍：

编写一个程序，开启 3 个线程，这三个线程的 ID 分别为 A、B、C，
每个线程将自己的 ID 在屏幕上打印 10 遍，要求输出的结果必须按顺序显示。
如：ABCABCABC…… 依次递归

代码实现：

```java
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

public class TestABCAlternate {

    public static void main(String[] args) {
        AlternateDemo ad = new AlternateDemo();
        new Thread(new Runnable() {
            @Override
            public void run() {
                for (int i = 0; i < 10; i++) {
                    ad.loopA();
                }
            }
        }, "A").start();
        new Thread(new Runnable() {
            @Override
            public void run() {
                for (int i = 0; i < 10; i++) {
                    ad.loopB();
                }
            }
        }, "B").start();
        new Thread(new Runnable() {
            @Override
            public void run() {
                for (int i = 0; i < 10; i++) {
                    ad.loopC();
                }
            }
        }, "C").start();
    }
}
class AlternateDemo {
    private int number = 1;//当前正在执行线程的标记

    private Lock lock = new ReentrantLock();
    private Condition condition1 = lock.newCondition();
    private Condition condition2 = lock.newCondition();
    private Condition condition3 = lock.newCondition();

    public void loopA() {
        lock.lock();
        try {

            if(number != 1) {
                condition1.await();
            }
            System.out.println(Thread.currentThread().getName());
            number = 2;
            condition2.signal();

        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }

    public void loopB() {
        lock.lock();
        try {

            if(number != 2) {
                condition2.await();
            }
            System.out.println(Thread.currentThread().getName());
            number = 3;
            condition3.signal();

        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }

    public void loopC() {
        lock.lock();
        try {
            if(number != 3) {
                condition3.await();
            }
            System.out.println(Thread.currentThread().getName());
            number = 1;
            condition1.signal();//唤醒线程1

        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }
}
```

解析：

线程是抢占式进行的，要按序交替，必须实现线程通信，就要用到等待唤醒。可以使用同步方法，也可以使用同步锁。



#### Read-Write 读写锁

写写/读写 需要互斥；读读不需要互斥

```java
import java.util.concurrent.locks.ReadWriteLock;
import java.util.concurrent.locks.ReentrantReadWriteLock;

public class TestReadWriteLock {

    public static void main(String[] args) {
        ReadWriteLockDemo rw = new ReadWriteLockDemo();
        new Thread(new Runnable() {
            @Override
            public void run() {
                rw.write((int) (Math.random() * 101));
            }
        }, "write").start();

        for(int i = 0; i < 100; i++) {
            new Thread(new Runnable() {
                @Override
                public void run() {
                    rw.read();
                }
            }).start();
        }

    }

}

class ReadWriteLockDemo {
    private int number = 0;

    private ReadWriteLock lock = new ReentrantReadWriteLock();//读写锁

    //读(可以多个线程同时操作)
    public void read() {
        lock.readLock().lock();
        try {
            System.out.println(Thread.currentThread().getName() + ": " + number);
        } finally {
            lock.readLock().unlock();
        }
    }
	//写(一次只能有一个线程操作)
    public void write(int number) {
        lock.writeLock().lock();
        try {
            System.out.println(Thread.currentThread().getName());
            this.number = number;
        } finally {
            lock.writeLock().unlock();
        }
    }

}

```



#### 线程八锁

