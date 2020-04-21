### <center>synchronized & ReentrantLock</center>

4. synchronized 1.6前是重量级锁，调用操作系统函数； 1.7后优化



ReentrantLock：

自旋：

```java 
volatile int status = 0;

void lock() {
	while(!CompareAndSet(0, 1)) { //如果status 为 0，将其改为 1，返回true，!true == false, 跳出循环
        
    }
}

void unlock() {
    status = 0;
}

boolean compareAndSet(int expect, int newValue) {
    //CAS操作，修改status成功则返回true
    if(status == expect) status = newValue; 
}
```

缺点：耗费cpu资源。没有竞争到锁的线程一直占用资源cas

解决思路：让得不到锁的线程让出cpu

yield + 自旋

```java
volatile int status = 0;
void lock() {
    while(!CompareAndSet(0, 1)) { //如果status 为 0，将其改为 1，返回true，!true == false, 跳出循环
        yield();//cpu调度，让出cpu
    }
}
void unlock() {
    status = 0;
}
```

问题：线程让出资源后，有可能下次还是选择该线程



sleep + 自旋

```java
volatile int status = 0;
void lock() {
    while(!CompareAndSet(0, 1)) { 
        sleep();
    }
}
void unlock() {
    status = 0;
}
```

缺点：睡眠时间无法确定



park + 自旋（ReentrantLock原理）

```java
volatile int status = 0;
Queue parkQueue;

void lock() {
    while(!compareAndSet(0, 1)) {
        park();//睡眠
    }
}
void unlock() {
    status = 0;
    lock_notify();
}

void park() {
    //将当前线程加入等待队列
    parkQueue.add(currentThread);
    //将当前线程释放cpu
    releasecpu
}
```



