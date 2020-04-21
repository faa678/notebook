#### ReentrantLock 加锁和释放锁的底层原理



ReentrantLock类中构造函数

```java
public class ReentrantLock implements Lock, java.io.Serializable {
    private final Sync sync;
	
    public ReentrantLock() {
        sync = new NonfairSync();//初始化Sync类象syn为内部类
    }

    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }
    
    ......
    
    public void lock() {
        sync.lock();//因为sync在构造函数中初始化为FairSync对象或NonfairSync对象，所以调用对应的lock函数
    }
}
```

其中ReentrantLock内部类NonfairSync

```java
static final class NonfairSync extends Sync {
    private static final long serialVersionUID = 7316153563782823691L;


    final void lock() {
        if (compareAndSetState(0, 1))//先尝试CAS修改state的值
            setExclusiveOwnerThread(Thread.currentThread());
        else
            acquire(1);//acquire最终调用tryAcquire
    }

    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }
}
```

nonfairTryAcquire方法：

```java
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {//如果没有线程持有锁
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {//如果当前线程持有锁
        //可重入锁
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    //如果其他线程持有锁，互斥
    return false;
}
```

相对的，公平锁的获取方法：

```java
/**
  * Fair version of tryAcquire.  Don't grant access unless
  * recursive call or no waiters or is first.
  公平的tryAcquire版本。除非递归调用或没有等待的任务或是第一个，否则不要授予访问权限。
*/
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}

public final boolean hasQueuedPredecessors() {
    // The correctness of this depends on head being initialized
    // before tail and on head.next being accurate if the current
    // thread is first in queue.
    Node t = tail; // Read fields in reverse initialization order
    Node h = head;
    Node s;
    //判断是否有多个任务
    return h != t &&	
        ((s = h.next) == null || s.thread != Thread.currentThread());
}
```



AQS对资源的共享方式：

* Exclusive(独占)：只有一个线程能执行，如ReentrantLock。又可分为公平锁和非公平锁：
  * 公平锁：按照线程在队列中的排队顺序，先到者先拿到锁
  * 非公平锁：当线程要获取锁时，无视队列顺序直接抢锁
* Share：多个线程可同时执行，如Semaphore，CountDownLatch，CyclicBarrier，ReadWriteLock。



ReentrantLock内部包含了一个AQS对象，是实现加锁和释放锁的核心组件

这个AQS对象内部有一个int类型核心变量state，代表加锁状态，初始状态下是0；

AQS对象内部还有一个关键变量，用来记录当前加锁的是哪个线程，初始状态下是null。



1. 线程1来调用ReentrantLock的lock方法尝试加锁（用CAS操作将state值从0变为1），当线程1加锁成功后，设置当前加锁线程是自己

![avatar](/home/faa/pictures/AQS1.png)

ReentrantLock是可重入锁，如果当前线程持有锁，可以多次加锁或释放锁。-----同一个线程

2. 锁互斥的实现

线程2来加锁，发现state是1，将state从0 变为1 的过程失败，检查是否自己加的锁，发现是线程1的锁，加锁失败。

![avatar](/home/faa/pictures/AQS2.png)

接着，线程2会进入AQS中的一个等待队列，等待线程1 释放锁。

![avatar](/home/faa/pictures/AQS3.png)

3. 接着，线程1在执行完自己的业务后，就会释放锁。将state递减1，如果state为0，彻底释放锁，并将加锁线程置为null

![avatar](/home/faa/pictures/AQS4.png)

4. 接下来，从队列队头唤醒线程2重新尝试加锁。

   ![avatar](/home/faa/pictures/AQS5.png)



对于CountDownLatch来说，任务分为N个子线程去执行，state也初始化为N。这N个子线程并行执行，每个子线程执行完后CountDown() 一次，state会CAS减 1 。等到所有子线程都执行完后（即state==0），会unpark()主调用线程，然后主调用线程就会从await()返回，继续后余动作。