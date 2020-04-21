### <center>ThreadLocal</center>

#### 1. ThreadLocal

##### 1.1 介绍

​	ThreadLocal 类用来提供线程内部的局部变量。这种变量在多线程环境下访问（通过 get 和 set 访问）时能保证各个线程内的变量相对独立于其他线程内的变量。ThreadLocal 实例通常都是 private static 类型的，用于关联线程和线程上下文。<font color=red>减少同一个线程内多个函数或组件之间一些公共变量传递的复杂度</font>。

1. 线程并发：多线程场景下
2. 传递数据：可以通过 ThreadLocal 在同一线程，不同组件中传递公共变量
3. 线程隔离：每个线程的变量都是独立的，不会互相影响



#### 1.2 基本使用

##### 1.2.1 常用方法

ThreadLocal()：构造函数

public void set(T value)：设置当前线程绑定的局部变量

public T get()：获取当前线程绑定的局部变量

public void remove()：移除当前线程绑定的局部变量



示例代码（未使用ThreadLocal）：

```java
public class TestThreadLocal {

    private String content;
    private String getContent() {
        return content;
    }

    private void setContent(String content) {
        this.content = content;
    }

    public static void main(String[] args) {
        TestThreadLocal ttl = new TestThreadLocal();

        for (int i = 0; i < 5; i++) {
            Thread thread = new Thread(new Runnable() {
                @Override
                public void run() {
                    ttl.setContent(Thread.currentThread().getName() + "的数据");
                    System.out.println("=============================");
                    System.out.println(Thread.currentThread().getName() + ttl.getContent());
                }
            });
            thread.setName("线程" + i);
            thread.start();
        }
    }
}

```

执行结果：

```
线程1线程2的数据
线程2线程2的数据
线程3线程3的数据
线程0线程3的数据
线程4线程4的数据
```

解析：

content是共享数据



示例代码2：

```java
ublic class TestThreadLocal {

    ThreadLocal<String> tl = new ThreadLocal<>();

    private String content;

    private String getContent() {
        return tl.get();//获取当前线程绑定的变量
    }

    private void setContent(String content) {
//        this.content = content;
        tl.set(content);//将content绑定到当前线程
    }
    public static void main(String[] args) {
        TestThreadLocal ttl = new TestThreadLocal();
        for (int i = 0; i < 5; i++) {
            Thread thread = new Thread(new Runnable() {
                @Override
                public void run() {
                    ttl.setContent(Thread.currentThread().getName() + "的数据");
                    System.out.println(Thread.currentThread().getName() + ttl.getContent());

                }
            });
            thread.setName("线程" + i);
            thread.start();
        }
    }
}
```

执行结果：

```
线程0线程0的数据
线程2线程2的数据
线程1线程1的数据
线程3线程3的数据
线程4线程4的数据
```

解析：

ThreadLocal：

1. set()：将变量绑定到当前线程中

 	2. get()：获取当前线程绑定的变量



##### 1.3 ThreadLocal 和 synchronized

1.3.1 

示例代码3：使用synchronized实现

```java
public class TestThreadLocal {

    private String content;
    
    private String getContent() {
        return content;
    }
    private void setContent(String content) {
        this.content = content;
//        tl.set(content);
    }
    public static void main(String[] args) {
        TestThreadLocal ttl = new TestThreadLocal();
        for (int i = 0; i < 5; i++) {
            Thread thread = new Thread(new Runnable() {
                @Override
                public void run() {
                    synchronized (this) {
                        ttl.setContent(Thread.currentThread().getName() + "的数据");
                        System.out.println(Thread.currentThread().getName() + ttl.getContent());
                    }
                }
            });
            thread.setName("线程" + i);
            thread.start();
        }
    }
}
```

执行结果：

```
线程0线程0的数据
线程1线程1的数据
线程3线程3的数据
线程2线程2的数据
线程4线程4的数据
```

解析：

​	加锁确实可以解决问题，但是这里强调的是线程数据隔离问题，而不是共享数据问题。



1.3.2 ThreadLocal 和 synchronized 的区别

|        | synchronized                                                 | ThreadLocal                                                  |
| ------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 原理   | 同步机制采用时间换空间的方式，只提供了一份变量，让不同的线程排队访问共享变量 | ThreadLocal采用以空间换时间，为每一个线程都提供了一份变量副本，从而实现线程数据隔离 |
| 侧重点 | 线程访问资源同步                                             | 线程数据隔离                                                 |



2. 运用场景

* 操作数据库转账。

AccountDao：

```java
void out(String outUser, int money) {
	sql = "money--";
    Connection conn = JdbcUtils.getConnection();
    
}
void in(String inUser, int money) {
    sql = "money++";
}
```

AccountService：

```java
boolean transfer(String outUser, String inUser, int money) {
	AccountDao ad = new AccountDao();
	Connection conn = null;
    
    try {
        conn = JdbcUtils.getConnection();
        conn.setAutoCommit();//设置手动提交事务
        
        ad.out(outUser, money);
        
        //模拟异常
        int i = 1 / 0;
        
        ad.in(inUser, money);
        JdbcUtils.commitAndClose(conn);
    }
}
```



* 当账户A转出成功，账户B转入失败时，就会造成数据不一致，此时可以使用数据库事务。为了保证所有的操作都在一个事务中，使用的连接必须是同一个

* 并发情况下，每个线程只能操作各自的连接
* 事务的使用注意点：
  * service层和dao层的连接对象保持一致（dao 的 out 和 in 接收一个connection对象）
  * 每个线程的connection对象必须前后一致，线程隔离

常规解决方案：

	1. 传参：将service层的connection对象直接传递到dao层
 	2. 加锁：事务代码块加 synchronized 锁

问题：

1. 提高代码耦合度
2. 降低性能

ThreadLocal 解决方案：

![avatar](/home/faa/pictures/ThreadLocal事务.png)



### 3. ThreadLocal 内部结构

每个 ThreadLocal 维护一个 ThreadLocalMap，这个 Map 的 key 是 ThreadLocal 实例本身，value 才是要存储的值 Object。

![avatar](/home/faa/pictures/ThreadLocal内部结构.png)

	* 每个 Thread 线程内部都有一个Map（ThreadLocalMap）
	* Map 里面存储 ThreadLocal 对象（key）和线程的变量副本（value）
	* Thread 内部的 Map 是由 ThreadLocal 维护的，由 ThreadLocal 负责向 map 获取和设置线程的变量值。
	* 对于不同的线程，每次获取副本之值时，别的线程并不能获取到当前线程的副本值，形成了副本的隔离。

与早期对比：

![avatar](/home/faa/pictures/ThreadLocal结构对比.png)

好处：

1. 每个Map存储的 Entry 数量变少
2. 当 Thread 销毁的时候，ThreadLocalMap 也会随之销毁，减少内存的使用



### 4. ThreadLocal 源码

| 方法声明                   | 描述                         |
| -------------------------- | ---------------------------- |
| protected T initialValue() | 返回当前线程局部变量的初始值 |
| public void set(T value)   | 设置当前线程绑定的局部变量   |
| public T get()             | 获取当前线程绑定的局部变量   |
| public void remove()       | 移除当前线程绑定的局部变量   |

从 Thread 类入手：

```java
public class Thread implements Runnable {
    ................    
	//与此线程有关的ThreadLocal值，由ThreadLocal类维护
	ThreadLocal.ThreadLocalMap threadLocals = null;
    
    //与此线程有关的InheritableThreadLocal值，由InheritableThreadLocal类维护
	ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
	.................
}
```

ThreadLocal 类中：

set 方法：

```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);//当前的 ThreadLocal 对象实例作为 key。
    else
        createMap(t, value);
}
```



get 方法：

```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    //两种情况
    //1. map 不存在，表示此线程没有维护的ThreadLocalMap对象
    //2. map 存在，但是没有与当前ThreadLocal关联的entry，即 e == null
    return setInitialValue();
}

private T setInitialValue() {
    //initialValue可以被子类重写，如果不重写返回默认null
    T value = initialValue();
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
    return value;
}

```

createMap方法：

```java
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

remove方法：

```java
public void remove() {
    ThreadLocalMap m = getMap(Thread.currentThread());
    if (m != null)
        m.remove(this);
}
```



initialValue 方法：返回线程局部变量的初始值

```java
protected T initialValue() {
    return null;
}
```



### 5. ThreadLocalMap源码

#### 5.1

ThreadLocalMap 是 ThreadLocal 的内部类，没有实现 Map 接口，用独立的方式实现了 Map 的功能，内部的 Entry 也是独立实现。

重要变量：

```java
//初始容量 16，必须为 2 的幂
private static final int INITIAL_CAPACITY = 16;

//entry 数组
private Entry[] table;

//entry 个数
private int size = 0;

//扩容阈值
private int threshold; // Default to 0
```

存储结构Entry：

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {// key 是 ThreadLocal 对象
        super(k);
        value = v;
    }
}
```

Entry 继承自 WeakReference，也就是 key（ThreadLocal）是弱引用，其目的是将ThreadLocal对象的生命周期和线程生命周期解绑。



#### 5.2 弱引用和内存泄漏

内存泄漏：

	* 内存溢出，没有足够的内存提供申请者使用
	* 内存泄漏：程序中已动态分配的堆内存由于某种原因程序未释放或无法释放，造成系统内存的浪费，导致程序运行速度减慢甚至系统崩溃，内存泄漏的堆积将导致内存溢出。

弱引用

​	java 中的引用有强、软、弱、虚。

* 强引用：常见的普通对象引用，只要还有强引用指向对象，就表示对象还活着，垃圾回收器就不会回收这种对象
* 弱引用：垃圾回收器一旦发现只具有弱引用的对象，不管当前内存空间是否足够，都会回收它的内存。

1. 如果 key 使用强引用： 无法避免内存泄漏

![avatar](/home/faa/pictures/假设ThreadLocalMap中使用强引用.png)

* 假设业务代码中使用完ThreadLocal，threadLocal Ref 被回收了
* 但是threadLocalMap的Entry强引用了 threadLocal ，造成 threadLocal 无法回收
* 在没有手动删除这个 Entry 以及 CurrentThread 依然运行的前提下，始终有强引用链 CurrentThreadRef -> currentThread -> threadLocalMap -> Entry，Entry 就不会被回收（Entry 中包括了 ThreadLocal 实例和 value），导致 Entry 内存泄漏

2. 如果 key 使用弱引用：也无法避免内存泄漏

![avatar](/home/faa/pictures/假设ThreadLocalMap中使用弱引用.png)

* 假设ThreadLocal，threadLocal Ref被回收了
* 由于 ThreadLocalMap 只持有 ThreadLocal 的弱引用，没有任何强引用指向 threadLocal 实例，所以 threadLocal 就可以顺利被 gc 回收，此时 Entry 中的 key == null
* 但是在没有手动删除这个 Entry 以及 CurrentThread 依然运行的前提下，也存在有强引用链 thread Ref -> currentThread -> threadLocalMap -> entry -> value，value 不会被回收，而这块value永远不会被访问到了，导致 value 内存泄漏。



出现内存泄漏的原因：

	* 没有手动删除这个 Entry
	* CurrentThread 依然运行

根本原因：ThreadLocalMap 的生命周期跟 Thread 一样长，如果没有手动删除对应的 key 就会导致内存泄漏。

所以，使用完 ThreadLocal 要及时remove，无论 key 是强引用还是弱引用都不会有问题。

所以为什么要使用弱引用：

​	在 ThreadLocalMap 中的 set/getEntry 方法中，会对 key 为 null（也即是 ThreadLocal 为 null）进行判断，如果为 null 的话，会将 value 置为 null。

​	这就意味着使用完 ThreadLocal，CurrentThread 依然运行的前提下，就算忘记调用 remove 方法，弱引用比强引用可以多一层保障：弱引用的 ThreadLocal 会被回收，对应的 value 在下一次 ThreadLocalMap 调用 set，get，remove 中的任一方法的时候会被清除（间接调用到 set/getEntry），从而避免内存泄漏。

​	

#### 5.3 ThreadLocalMap 中 hash 冲突的解决

1. ThreadLocalMap构造方法：

```java
/*
	firstKey：当前 ThreadLocal 实例
	firstValue：要保存的线程本地变量
*/
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
    table = new Entry[INITIAL_CAPACITY];
    int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
    table[i] = new Entry(firstKey, firstValue);
    size = 1;
    setThreshold(INITIAL_CAPACITY);
}
```

setThreshold方法：

```java
private void setThreshold(int len) {
    threshold = len * 2 / 3;
}
```



2. 重点分析：

```java
int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
```

a. 关于 firstKey.threadLocalHashCode：

```java
private final int threadLocalHashCode = nextHashCode();

private static int nextHashCode() {
	return nextHashCode.getAndAdd(HASH_INCREMENT);
}

private static AtomicInteger nextHashCode = new AtomicInteger();

private static final int HASH_INCREMENT = 0x61c88647;
```

HASH_INCREMENT = 0x61c88647 跟黄金分割数有关，目的是为了让 hash 码均匀分布在 2 的 n 次方的数组里，尽量避免哈希冲突

b. 关于  & (INITIAL_CAPACITY - 1)

hashCode & (size - 1) 相当于取模的高效运算要求 size 必须是 2 的整次幂，这也能保证在索引不越界的前提下，使得 hash 发生冲突的次数减少。

3. ThreadLocalMap 中的 set 方法

```java
private void set(ThreadLocal<?> key, Object value) {

    // We don't use a fast path as with get() because it is at
    // least as common to use set() to create new entries as
    // it is to replace existing ones, in which case, a fast
    // path would fail more often than not.

    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);

    for (Entry e = tab[i]; e != null; e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();//return this.referent
		
        //ThreadLocal 对应的 key 存在，直接覆盖之前的值
        if (k == key) {
            e.value = value;
            return;
        }
		//key 为 null，但是值不为 null，说明之前的 ThreadLocal 对象已经被回收了
        //当前数组中的 Entry 是一个陈旧的(stable)的元素
        if (k == null) {
            //用新元素替换旧元素
            replaceStaleEntry(key, value, i);
            return;
        }
    }

    //table[i] 为空时
    tab[i] = new Entry(key, value);
    int sz = ++size;
    //cleanSomeSlots 清理 key 为 null 的 Entry
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}

private static int nextIndex(int i, int len) {
    //环形数组
    return ((i + 1 < len) ? i + 1 : 0);//线性探测法，和 HashMap 不一样
}
```