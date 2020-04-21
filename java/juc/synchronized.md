### <center>synchronized关键字</center>

synchronized底层用jvm实现的关键字，用来控制线程同步，是一种同步锁。

####  synchronized作用：

1. 原子性：确保线程互斥地访问同步代码

2. 可见性：

   * 线程加锁前，将清空工作内存中共享变量的值，使用到共享变量时，从主存中获取最新的共享变量的值

   * 线程解锁前，必须把共享变量的最新值刷新到主存中

   ![avatar](/home/faa/pictures/论文/synchronized保证原子性.png)

3. 有序性：不存在依赖关系的操作可能会发生重排序。加synchronized后，依然会发生重排序，但是同步代码块可以保证只有一个线程执行同步代码中的代码。

#### synchronized应用

synchronized可以把任何一个非null对象作为"锁"，在 HotSpot JVM 实现中，锁有个专门的名字：<font color=blue>对象监视器</font>

 1. 修饰普通方法

    作用于当前实例对象加锁，进入同步代码前要获得<font color=red>当前实例对象</font>锁，锁就是当前实例对象

 2. 修饰静态方法

    作用于当前类的所有对象加锁，进入同步代码前要获得当前<font color=red>Class对象</font>锁，锁就是Class对象

 3. 修饰代码块

    锁是括号里面的对象，对给定对象加锁，进入同步代码库前要获得给定对象的锁。

    * synchronized(A.class) {   }：对Class对象加锁
    * synchronized(任意对象) {   }：对对象加锁



* synchronized同步代码块的实现使用的是 <font color=red>monitorenter</font> 和 <font color=red>monitorexit</font> 指令，执行 monitorenter 指令时，线程试图获取锁也就是 monitor，monitor对象存在于每个java对象头。

* synchronized修饰方法使用的是 <font color=red>ACC_SYNCHRONIZED</font> 标识。



示例代码1： 没有同步的情况

```java
public class TestSyn implements Runnable {

    private static int syn = 0;

    public void increase() {
        syn++;
    }

    @Override
    public void run() {
        for(int i = 0; i < 1000; i++) {
            increase();
        }
    }

    public static void main(String[] args) throws InterruptedException {
        TestSyn ts = new TestSyn();
        Thread t1 = new Thread(ts);
        Thread t2 = new Thread(ts);
        t1.start();
        t2.start();
        t1.join();
        t2.join();
        Thread.sleep(5000);
        System.out.println(syn);
    }
}
```

执行结果：（可能结果）

```
1974
```

解析：

没有进行同步的情况下，有可能线程1和线程2同时进入 increase 方法并抢占到共享值，自增后写回主存。



示例代码2：修饰实例方法

```java
public class TestSyn implements Runnable {

    private static int syn = 0;

    public synchronized void increase() {//使用synchronized修饰实例方法
        syn++;
    }

    @Override
    public void run() {
        for(int i = 0; i < 1000; i++) {
            increase();
        }
    }

    public static void main(String[] args) throws InterruptedException {
		//实例化一个对象
        TestSyn ts = new TestSyn();
        Thread t1 = new Thread(ts);
        Thread t2 = new Thread(ts);
        t1.start();
        t2.start();
        t1.join();
        t2.join();
        Thread.sleep(5000);
        System.out.println(syn);
    }
}
```

执行结果：

```
2000
```

解析：

main函数中只实例化了一个TestSyn对象，两个线程运行的时候同一时刻只能有一个线程获取到对象的锁。当以个线程获取了该对象的锁之后，其他线程无法获取该对象的锁，也无法访问该对象的其他synchronized实例方法，但是可以访问该对象的非synchronized方法。



如果，将上面修改如下，为每个线程创建一个对象实例：

```java
    public static void main(String[] args) throws InterruptedException {
        //实例化对象
        TestSyn ts1 = new TestSyn();
        TestSyn ts2 = new TestSyn();
        Thread t1 = new Thread(ts1);
        Thread t2 = new Thread(ts2);
        t1.start();
        t2.start();
        Thread.sleep(2000);
        System.out.println(syn);
    }
```

此时的运行结果可能会出现错误。因为t1和t2抢占的不是同一个锁。t1抢占的ts1锁；t2抢占的是ts2锁。



示例代码3：修饰静态方法，修改increase方法为static，并且为每一个线程创建一个对象实例

```java
    public synchronized void increase() {//使用synchronized修饰实例方法
        syn++;
    }
    public static void main(String[] args) throws InterruptedException {
        //实例化对象
        TestSyn ts1 = new TestSyn();
        TestSyn ts2 = new TestSyn();
        Thread t1 = new Thread(ts1);
        Thread t2 = new Thread(ts2);
        t1.start();
        t2.start();
        Thread.sleep(2000);
        System.out.println(syn);
    }
```

执行结果：

```
2000
```

解析：

synchronized修饰静态方法，不管实例化多少个对象实例，锁对象都是当前类对象，被同步的方法同一时刻只能被一个线程执行。



示例代码4：

```java
public class TestSyn implements Runnable {

    private static int syn = 0;

    @Override
    public void run() {
        synchronized (this) {//synchronized给当前对象加锁
            for(int i = 0; i < 10000; i++) {
                syn++;
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {

        TestSyn ts = new TestSyn();
        Thread t1 = new Thread(ts);
        Thread t2 = new Thread(ts);
        t1.start();
        t2.start();
        t1.join();
        t2.join();
//        Thread.sleep(5000);
        System.out.println(syn);
    }
}
```

执行结果：

```
2000
```

解析：

每次当线程进入synchronized包裹的代码块时就会要求当前线程持有当前对象锁，其他的线程就必须等待，保证了同步。



#### synchronized特性

1. 可重入特性：一个线程可以多次执行synchronized，重复获取同一把锁

示例：在线程类的 run() 方法中使用嵌套的同步代码块

```java
public class Demo01 {

    public static void main(String[] args) {
        new MyThread().start();
        new MyThread().start();
    }

}
class MyThread extends Thread {
    @Override
    public void run() {
        synchronized ("") {
            System.out.println(getName() + "进入了同步代码块1");
            try {
                Thread.sleep(5000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            synchronized ("") {
                System.out.println(getName() + "进入了同步代码块2");
            }
        }
    }
}
```

执行结果：

```
Thread-0进入了同步代码块1
Thread-0进入了同步代码块2
Thread-1进入了同步代码块1
Thread-1进入了同步代码块2
```

解析：

线程1获取第一个锁，锁对象头会有锁计数器，线程1进入第二个代码块后，计数器 == 2，逐渐减1释放锁。

好处：

可以避免死锁

封装代码

2. 不可中断性

   一个线程获得锁后,另一个线程想要获得锁,必须处于阻塞或等待状态,如果第一个线程不释放锁,第二个线程会一直阻塞或等待,不可被中断（interrupt）。

示例代码1：synchronized不可中断

```java
/*
目标:演示synchronized不可中断
1.定义一个Runnable
2.在Runnable定义同步代码块
3.先开启一个线程来执行同步代码块,保证不退出同步代码块
4.后开启一个线程来执行同步代码块(阻塞状态)
5.停止第二个线程
*/
public class Demo02 {
    private static Object obj = new Object();
    public static void main(String[] args) throws InterruptedException {
// 1.定义一个Runnable
        Runnable run = () -> {
// 2.在Runnable定义同步代码块
            synchronized (obj) {
                String name = Thread.currentThread().getName();
                System.out.println(name + "进入同步代码块");
// 保证不退出同步代码块
                try {
                    Thread.sleep(12000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        };
// 3.先开启一个线程来执行同步代码块
        Thread t1 = new Thread(run);
        t1.start();
        Thread.sleep(1000);
// 4.后开启一个线程来执行同步代码块(阻塞状态)
        Thread t2 = new Thread(run);
        t2.start();
// 5.停止第二个线程
        System.out.println("停止线程前");
        t2.interrupt();//synchronized是不可中断的
        System.out.println("停止线程后");
        System.out.println(t1.getName() + ": " + t1.getState());
        System.out.println(t2.getName() + ": " + t2.getState());//blocked阻塞状态而不是终止状态
    }
}
```

执行结果：

```
Thread-0进入同步代码块
停止线程前
停止线程后
Thread-0: TIMED_WAITING
Thread-1: BLOCKED
```

解析：

线程等待synchronized时是不可中断的。

示例代码2：ReentrantLock对象.lock()不可中断

```java
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;/*
目标:演示Lock不可中断和可中断
*/
public class TestSyn {
    private static Lock lock = new ReentrantLock();
    public static void main(String[] args) throws InterruptedException {
 		test01();
    }

    // 演示Lock不可中断
    public static void test01() throws InterruptedException {
        Runnable run = () -> {
            String name = Thread.currentThread().getName();
            try {
                lock.lock();
                System.out.println(name + "获得锁,进入锁执行");
                Thread.sleep(12000);
            } catch (InterruptedException e) {e.printStackTrace();
            } finally {
                lock.unlock();
                System.out.println(name + "释放锁");
            }
        };
        System.out.println("停止t2线程前");
        Thread t1 = new Thread(run, "线程1");
        t1.start();
        Thread.sleep(1000);//为了确保t2在t1后执行
        System.out.println(t1.getName() + ": " + t1.getState());
        Thread t2 = new Thread(run, "线程2");
        t2.start();
        Thread.sleep(1000);
        System.out.println(t2.getName() + ": " + t2.getState());
        t2.interrupt();
        System.out.println("停止t2线程后");
        Thread.sleep(1000);
        System.out.println(t1.getName() + ": " + t1.getState());
        System.out.println(t2.getName() + ": " + t2.getState());
    }
}
```

执行结果：

```
停止t2线程前
线程1获得锁,进入锁执行
线程1: TIMED_WAITING
线程2: WAITING
停止t2线程后
线程1: TIMED_WAITING
线程2: WAITING
线程1释放锁
线程2获得锁,进入锁执行
线程2释放锁

BUILD SUCCESSFUL in 13s
2 actionable tasks: 1 executed, 1 up-to-date
java.lang.InterruptedException: sleep interrupted
	at java.lang.Thread.sleep(Native Method)
	at com.faa.juc.syn.TestSyn.lambda$test01$1(TestSyn.java:60)
	at java.lang.Thread.run(Thread.java:748)
```

解析：

1. 线程1获得锁，并睡眠，所以线程1此时状态为 `TIMED_WAITING`

2. 此时线程2来抢占锁，但是线程1未释放锁，所以线程2抢占失败，状态为 WAITING
3. 执行t2.interrupt()，此时线程1还未睡眠完成，线程2状态仍然为WAITING，说明中断失败
4. 线程1睡眠完成，释放锁，此时线程2获得锁，进入锁执行
5. 线程2释放锁

ps：报错是睡眠被中断，暂时忽略



示例代码3：ReentrantLock对象.tryLock()可中断

```java
    public static void test02() throws InterruptedException {
        Runnable run = () -> {
            String name = Thread.currentThread().getName();
            boolean b = false;
            try {
                b = lock.tryLock(3, TimeUnit.SECONDS);
                if (b) {
                    System.out.println(name + "获得锁,进入锁执行");
                    Thread.sleep(12000);
                } else {
                    System.out.println(name + "在指定时间没有得到锁做其他操作");
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                if (b) {
                    lock.unlock();
                    System.out.println(name + "释放锁");
                }
            }
        };
        System.out.println("停止t2线程前");
        Thread t1 = new Thread(run, "线程1");
        t1.start();
        Thread.sleep(1000);//为了确保t2在t1后执行
        System.out.println(t1.getName() + ": " + t1.getState());
        Thread t2 = new Thread(run, "线程2");
        t2.start();
        Thread.sleep(1000);//为了使t2有足够时间从Runnable状态变成TIMED_WAITING状态
        System.out.println(t2.getName() + ": " + t2.getState());
        Thread.sleep(5000);//为了超过线程2等待的3秒
        t2.interrupt();
        System.out.println("停止t2线程后");
        Thread.sleep(1000);
        System.out.println(t1.getName() + ": " + t1.getState());
        System.out.println(t2.getName() + ": " + t2.getState());
    }
```

执行结果：

```
停止t2线程前
线程1获得锁,进入锁执行
线程1: TIMED_WAITING
线程2: TIMED_WAITING
线程2在指定时间没有得到锁做其他操作
停止t2线程后
线程1: TIMED_WAITING
线程2: TERMINATED
线程1释放锁
```

解析：

1. 线程1获得锁，并睡眠，所以线程1此时状态为 `TIMED_WAITING`

2. 此时线程2来抢占锁3秒，所以线程2状态为 TIMED_WAITING，超过3秒放弃抢占
3. 执行t2.interrupt()，此时线程1还未睡眠完成，线程2状态仍然变成TERMINATED，说明中断成功
4. 线程1睡眠完成，释放锁