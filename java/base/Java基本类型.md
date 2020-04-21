#### Java基础



[参考资料](https://juejin.im/post/5e18879e6fb9a02fc63602e2#heading-41)



Java基本类型

![avtar](/home/faa/pictures/java基本类型.png)

1. 面向对象与面向过程的区别

   * 面向过程：性能高
   * 面向对象：易维护、易复用、易扩展。
     * 封装、继承、多态

   Java性能相对差的原因主要是因为Java是<font color=red>半编译语言</font>，java code -> 字节码 -> 机器码

   

2. Java特点

   > - 面向对象（封装、继承、多态）
   > - 跨平台
   > - 支持多线程（C++没有内置的多线程机制，必须调用操作系统的多线程功能，C++11引入多线程）
   > - 编译与解释并存



3. JVM、JDK、JRE

   > JVM是运行Java字节码的虚拟机

   ![avtar](/home/faa/pictures/java code.png)

   > JDK：java sdk 
   >
   > JRE：java runtime



4. Java 和 C++ 的区别

   - 都是面向对象，都支持继承、封装、多态（C++支持面向对象也支持面向过程，Java纯面向对象）

   - Java不提供指针来直接访问内存，程序内存更加安全。C++可以通过指针直接访问到所指向的对象，Java都是通过对象引用如：<font color=blue>Object o = new Object();</font> new出来的对象不能直接被访问到

   - Java类<font color=red>单继承</font>，C++支持多继承。Java接口可以有多个实现

   - Java有自动内存管理机制

   - Java字符串没有'\0'作为结束符，C有。因为Java字符串是一个对象，有length属性，所以没有必要标识字符串的结束位置。

     

5. 字符常量和字符串常量

   - 形式上： 'a', "abc"

   - 含以上：字符常量 -> ASCII值，可以参加表达式运算；字符串常量代表字符串在内存中的地址值

   - Java中char占<font color=red>2个</font>字节；字符串若干

     

6. 构造器（构造函数）

   不能被重写（override），可以被重载（overload）

   

7. 重载与重写
   
   [重载与重写](/home/faa/files/notebook/java/jvm/字节码/03.方法字节码.md)
   
   - 重载：同一个类中。Java允许重载任何方法
     - 相同部分：方法名
     - 不同部分：参数类型  or  参数个数  or  参数顺序
     - 可以不同：返回值、访问修饰符（不能有两个名字相同、参数类型也相同但是返回类型不同的方法）
   - 重写：子类对父类允许访问的方法的实现过程的重新编写。
     - 参数列表相同
     
     - 返回值范围 <= 父类
     
     - 异常范围 <= 父类
     
     - 访问修饰符范围 >= 父类
     
     - 父类中private方法不能被重写，也不能被子类访问
     
       

8. java特性

   * 封装：对象属性私有化，提供给外界访问属性的方法

   * 继承：

   * 多态：一个引用变量指向哪个类的实例变量，该引用变量发出的方法调用到底是哪个类中实现的方法，运行期才能确定。

     

9. String、StringBuffer和StringBuilder

   9. * 可变性：

        * String：private <font color=red>final</font> char[] value, <font color=red>不可变</font>
        * StringBuilder、StringBuffer：都继承自AbstractStringBuilder类，在AbstractStringBuilder类中使用char[] value, 但是<font color=blue>没有final</font>修饰，所以两种对象都是<font color=blue>可变</font>的

        AbstractStringBuilder.java

        ```java
        abstract class AbstractStringBuilder implements Appendable, CharSequence {
        	char[] value;
        	
        	int count;
        	
        	AbstractStringBuilder(int capacity) {
        		value = new char[capacity];
        	}
        }
        ```

      * 线程安全性：

        * String中的对象不可变，可以理解为常量，<font color=blue>线程安全</font>
        * String<font color=blue>Buffer</font>对方法加了同步锁或者堆调用的方法加了同步锁，所以<font color=blue>线程安全</font>
        * String<font color=red>Builder</font>没有对方法进行加同步锁，所以是<font color=red>非线程安全</font>的

      * 性能：
        * 每次对String类型进行改变的时候，都会<font color=red>生成一个新的String对象</font>，然后将指针指向新的String对象
        * StringBuffer和StringBuilder每次都会对<font color=red>对象本身</font>进行操作

      * 总结：
        * 少量数据：String
        * 单线程操作字符串缓冲区下操作大量数据：适用StringBuilder
        * 多线程操作字符串缓冲区下操作大量数据：适用StringBuffer

   

10. 静态方法内调用非静态成员

    * 是非法的。

    * 静态方法是属于类的，非静态方法属于类的实例对象，在<font color=red>类加载</font>的时候就会分配内存，可以通过类名直接去访问；

    * 非静态成员属于类的对象，所以只有在类的对象实例化之后才存在，然后通过类的对象去访问。

    

11. <font color=purple>Java中定义一个无参构造方法的作用</font>

    Java在执行子类的构造方法之前，如果没有用super()来调用父类特定的构造方法，则会调用父类中的无参构造方法。因此，如果父类中之定义了有参数的构造方法（不会有默认构造方法），而在子类中又没有用super()来调用弗雷中特定的构造方法，则编译时将发生错误。



12. 接口和抽象类

    * 接口方法默认public，方法在接口中不能有实现（Java8可以有默认实现）

    * 接口中都是static final

    * 接口默认方法修饰符是public，抽象不能是private

      

13. 成员变量和局部变量

    *  语法上，成员属于类
    * 成员可以public、private、protect、static等
    * 存储方式上，如果成员变量是static的，属于类，不是static的话，属于实例
    * 成员变量会自动赋初值

14. 对象实体与对象引用
    
* 实体放在方法区，引用在虚拟机栈，引用指向实体
  
15. new和构造方法作用    [详情参考](https://www.cnblogs.com/xwb583312435/p/8672622.html)

    用new创建并初始化对象步骤：

    1、给对象的实例变量（非“常量”）分配内存空间，默认初始化成员变量；

    2、成员变量声明时的初始化

    3、初始化块初始化（又称为构造代码块或非静态代码块）；

    4、构造方法初始化

16. 静态方法和实例方法
    * 静态方法：类名.方法名 或 对象名.方法名
    * 实例方法：对象名.方法名
    * 静态方法在访问本类的成员时，只允许访问静态成员（静态成员变量和静态方法），不允许访问实例成员变量和实例方法；实例方法无此限制
17. == 和 equals
    * == ：判断对象<font color=red>内存地址</font>是否相等
    * equals ：String、Map、Integer、Double等的equals方法是被重写过的，比较的是<font color=red>值</font>；Object的equals比较的是对象的内存地址
    * 创建String对象时，虚拟机会在常量池中查找有没有已经存在的值和要创建的值相同的对象，如果有，就把他赋给当前对象；如果没有就在常量池中重新创建一个String对象

18. hashCode 和 equals [详情参考](https://blog.csdn.net/zj15527620802/article/details/88547914)

19. Java只有值传递  [详情参考1](https://blog.csdn.net/bjweimengshu/article/details/79799485) [详情参考2](https://blog.csdn.net/qq_35109096/article/details/81105320?depth_1-utm_source=distribute.pc_relevant.none-task&utm_source=distribute.pc_relevant.none-task)

    调用函数时，传递给参数的实参的地址的拷贝（如果实参在栈中，则直接拷贝该值），也就是都是拷贝的栈中的数据

    一般说的引用传递，在实际中传递的不过是引用对象的地址值（还是值传递）

    值传递和引用传递的区别不是传递的内容，而是实参到底有没有被复制一份给形参。在判断实参内容有没有受影响的时候，要看传递的是什么，如果传的是地址，就看地址的变化会不会有影响，而不是看地址指向的对象的变化。

20. 进程、线程

    进程在执行过程中可以产生多个线程。

    与进程不同的是同类的多个线程共享同一块内存空间和一组系统资源。

    进程是相互独立的。是系统运行程序的基本单位，是动态的。属于操作系统的范畴

21.线程基本状态

	* new：初始状态，线程被构建，但是还没有调用start()方法
	* runnable：运行状态（包括操作系统中的就绪和运行）
	* blocked：阻塞。线程阻塞于锁
	* waiting：等待。等待其他线程动作（通知或中断）
	* time_waiting：超时等待。可以在指定的时间自行返回
	* terminated：终止。线程执行完毕。

![avatar](/home/faa/pictures/线程状态.png)

22. final关键字
    * 修饰基本类型，数值不能更改；修饰引用类型，指向不能更改
    * 修饰类，不能被继承，类中成员方法隐式final
    * 修饰方法，锁定方法，防止继承类修改他；类中所有private方法都隐式final

23. 异常

    ![avatar](/home/faa/pictures/异常和错误.png)

* Throwable类

  常用方法：

  	* public String getMessage()：异常简要概述
  	* toString()：详细信息
  	* getLocalizedMessage()：异常对象本地化信息，需要覆盖，否则同getMessage()
  	* printStackTrace()：控制台打印Throwable对象封装的异常信息

* Error

  程序无法处理

* Exception

  城促本身可以处理

  

* 异常处理

  finally不会被执行的4种情况：

  * finally语句的第一行异常
  * 前面代码调用了System.exit(int)
  * 程序所在线程死亡
  * 关闭cpu

* try和finally都有return时，在方法返回之前，finally语句的内容将被执行，并且finally语句的返回值将会覆盖原始返回值。

  ```java
  public static int f(int value) {
  	try {
  		return value * value;
  	}
  	finally {
  		if(value == 2) {
  			return 0;
  		}
  	}
  }
  ```

  如果调用f(2)，返回值是0。

24. transient关键字  [详情参考](https://blog.csdn.net/yaomingyang/article/details/79321939)

    作用：阻止变量序列化；当对象被反序列化时，被transient修饰的变量不会被持久化和恢复.

    只能修饰变量，不能修饰类和方法

25. BIO（blocking I/O）、NIO(new io/non-blocking io)、AIO(NIO2/Asynchronous io)  [详情参考1](https://blog.csdn.net/qq_28323595/article/details/88909278) [[详情参考2](https://blog.csdn.net/ty497122758/article/details/78979302) [详情参考3](https://my.oschina.net/u/3471412/blog/2966696)

    BIO：同步阻塞io。数据的读取和写入必须组塞在一个线程内等待其完成。适用于活动链接数不是特别高。让每一个连接专注于自己的io。一个连接一个线程

    NIO：同步非阻塞。支持面向缓冲，基于通道。多路复用。NIO最重要的地方时当一个连接创建后，不需要对应一个线程，这个连接会被注册到多路复用器上，所有的连接只需要一个线程就可以搞定。当这个多路复用器发现连接上有请求的时候，才会开启一个线程进行处理。也就是一个请求一个线程。

    AIO：异步io。基于事件和回调机制。应用操作之后直接返回。后台处理完成，操作系统通知相应的线程进行后续操作。

<table><tr style="font-weight:bold;"><td width="120">组合方式</td><td>性能分析</td></tr><tr><td>同步阻塞</td><td>最常用的一种用法，使用也是最简单的，但是 I/O 性能一般很差，CPU 大部分在空闲状态。</td></tr><tr><td>同步非阻塞</td><td>提升 I/O 性能的常用手段，就是将 I/O 的阻塞改成非阻塞方式，尤其在网络 I/O 是长连接，同时传输数据也不是很多的情况下，提升性能非常有效。 这种方式通常能提升 I/O 性能，但是会增加CPU 消耗，要考虑增加的 I/O 性能能不能补偿 CPU 的消耗，也就是系统的瓶颈是在 I/O 还是在 CPU 上。</td></tr><tr><td>异步阻塞</td><td>这种方式在分布式数据库中经常用到，例如在网一个分布式数据库中写一条记录，通常会有一份是同步阻塞的记录，而还有两至三份是备份记录会写到其它机器上，这些备份记录通常都是采用异步阻塞的方式写 I/O。异步阻塞对网络 I/O 能够提升效率，尤其像上面这种同时写多份相同数据的情况。</td></tr><tr><td>异步非阻塞</td><td>这种组合方式用起来比较复杂，只有在一些非常复杂的分布式情况下使用，像集群之间的消息同步机制一般用这种 I/O 组合方式。如 Cassandra 的 Gossip 通信机制就是采用异步非阻塞的方式。它适合同时要传多份相同的数据到集群中不同的机器，同时数据的传输量虽然不大，但是却非常频繁。这种网络 I/O 用这个方式性能能达到最高。</td></tr></table>

26. Hashtable、Hashmap、ConcurrentHashMap  [详情参考](https://yuanrengu.com/2020/ba184259.html)  [详情参考](https://www.jianshu.com/p/cb2a801c45e3)

    * HashMap继承自AbstractMap类，而HashTable继承自Dictionary。但是都同时实现了map、Cloneable、Serializable三个接口。

    * HashSet基于HashMap实现。
    
    * HashTable的key、value都不能为null；HashMap的key、value可以，但是只能有一个key为null，但是可以有多个null的value；TreeMap的key、value都不能为null; ConcurrentHashMap的key和value都不能为null
    
    * HashMap<font color=red>非</font>synchronized，<font color=red>非线程安全</font>

    * HashTable是synchronized，线程安全,需要进行同步，所以单线程环境下速度比HashMap慢
    
    * HashTable和ConcurrentHashMap都可以用于多线程环境，但是和Collections.synchronizedMap(<Map<K, V> m>)方法获取的线程安全的Map一样，HashTable仅有单个锁，对整个集合加锁。所以当HashTable的大小增到一定程度的时候，性能会急剧下降，因为迭代时整个集合需要被锁定更长的时间。
    
    * 初始容量大小和每次扩充容量大小：
    
      * 创建时不指定：HashTable默认11，扩容2n+1；HashMap默认16，扩容2倍。
    
      * 创建时指定：HashTable直接使用；HashMap扩充至2的幂次方大小（tableSizeFor()）
    
      * HashMap带有初始容量构造函数：
    
        ```java
        public HashMap(int initialCapacity, float loadFactor) {
                if (initialCapacity < 0)
                    throw new IllegalArgumentException("Illegal initial capacity: " +
                                                       initialCapacity);
                //MAXIMUM_CAPACITY = 1 << 30
                //大于MAXIMUM_CAPACITY的初始值，重置为MAXIMUM_CAPACITY
                if (initialCapacity > MAXIMUM_CAPACITY)
                    initialCapacity = MAXIMUM_CAPACITY;
                if (loadFactor <= 0 || Float.isNaN(loadFactor))
                    throw new IllegalArgumentException("Illegal load factor: " +
                                                       loadFactor);
                this.loadFactor = loadFactor;
                this.threshold = tableSizeFor(initialCapacity);
            }
        ```
        
    
    tableSizeFor()保证HashMap总是用2的幂作为哈希表的大小：
        
    ```java
        static final int tableSizeFor(int cap) {
            int n = cap - 1;
            // >>> 无符号右移，空位以0补齐
            n |= n >>> 1;
            n |= n >>> 2;
            n |= n >>> 4;
            n |= n >>> 8;
            n |= n >>> 16;
            return (n < 0) ? 1 : (n >= MAXMUM_CAPAcITY) ? MAXMUM_CAPAcITY : n + 1;
        }
    ```
    
    * HashMap通过key的hashCode经过扰动函数(hash)得到hash值，通过（length - 1）& hash判断当前元素存放的位置
    
      hash方法：
    
      ```java
      static final int hash(Object key) {
      	int h;
          return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
      }
      ```
    
    * HashMap长度2的n次幂
    
      为了加快哈希计算以及减少哈希冲突。
    
      为了找到key的位置在哈希表的哪个槽，需要计算hash(key) % length，但是 % 比  & 慢很多，所以用&代替%，为了保证&的计算结果等于%的结果，需要把length - 1，也就是hash(key) & (length - 1), 也就是说hash%length == hash&(length - 1)的前提是length是2的n次幂
    
      length为偶数时，length - 1为奇数，保证了hash&(len - 1)的最后一位可能是0，也可能是1，可以保证散列的均匀。
    
      如果length为奇数，只会被散列到偶数下标位置，浪费空间。
    
      因此，length取2的整数次幂，为了使不同的hash值发生碰撞的概率较小，均匀散列
    
27. ArrayList 和 vector
    * ArrayList 通过数组实现，vector也是
    * vector支持线程同步，使用了synchronized，某一时刻只有一个线程能写vector，访问vector的话耗费时间

28. HashSet检查重复

    加入HashSet时，先计算对象的hashCode是否有，如果有，调用equals()方法检查hashCode相等的对象是否真的相同。

29. jdk1.8前HashMap多线程导致死循环的问题  [详情参考](https://coolshell.cn/articles/9606.html)

    并发下rehash会造成元素之间形成循环链表。

30. ConcurrentHashMap 和 HashTable

    * ConcurrentHashMap实现线程安全的方式：
      * jdk1.7时，ConcurrentHashMap（分段锁）对整个桶数组进行了分割分段（Segment），每一把锁只锁容器其中一部分数据，多线程访问容器里不同数据段的数据，就不会存在锁竞争，提高并发访问率。

        ![avatar](/home/faa/pictures/jdk1.7ConcurrentHashMap锁.png)

      * jdk1.8摒弃Segment，直接用Node数组+链表+红黑树的数据结构实现，并发控制使用synchronized和CAS来操作。

        ![avatar](/home/faa/pictures/jdk1.8ConcurrentHashMap.png)

    * HashTable实现线程安全的方式：

      * 使用synchronized对全表加锁。当一个线程访问同步方法，其他线程也访问时，可能会进入阻塞或轮询，效率低下。

        ![avatar](/home/faa/pictures/HashTable全表锁.png)

31. <font color=red>ConcurrentHashMap</font>

    实现线程安全：

    * jdk1.7:将数据分为一段一段的存储，然后每一段配一把锁，当一个线程占用锁访问其中一个段数据时，其他段的数据也能被其他线程访问。
    
      ![avatar](/home/faa/pictures/jdk1.7ConcurrentHashMap锁.png)
    
      ​		Segment实现了ReetrantLock,所以Segment是一种可重入锁。		
    
      ```java
      static class Segment<K, V> extends ReetrantLock implements Serializable {
      }
      ```
    
      每个HashEntry是一个链表结构的元素，每个Segment守护一个HashEntry数组里的元素，当对HashEntry数组的数据进行修改时，必须首先获得对应的Segment锁。
    
      
    
    * jdk1.8:取消Segment分段，采用CAS和synchronized保证并发安全。
    
      ![avatar](/home/faa/pictures/jdk1.8ConcurrentHashMap.png)
    
      synchronized只锁定当前链表或红黑树的首节点，这样只要hash不冲突，就不会产生并发。

32. Comparable 和 Comparator
    * java.lang.Comparable，使用compareTo(Object o)排序
    * java.util.Comparator，使用compare(Object o1, Object o2)排序

Comparable：

```java
package com.faa.base;

import java.util.Set;
import java.util.TreeMap;

public class MyTest3 implements Comparable<MyTest3>{
    private String name;
    private int age;



    @Override
    public int compareTo(MyTest3 o) {//
        return this.age - o.age;
    }

    public MyTest3(String name, int age) {
        super();
        this.name = name;
        this.age = age;
    }

    public static void main(String[] args) {
        TreeMap<MyTest3, String> pdata = new TreeMap<MyTest3, String>();
        pdata.put(new MyTest3("张三", 7), "zhangsan");
        pdata.put(new MyTest3("李四", 20), "lisi");
        pdata.put(new MyTest3("王五", 10), "wangwu");
        pdata.put(new MyTest3("小红", 5), "xiaohong");

// 得到key的值的同时得到key所对应的值
        Set<MyTest3> keys = pdata.keySet();
        for (MyTest3 key : keys) {
            System.out.println(key.age + "-" + key.name);
        }
    }

}

```

Comparator：

```java
package com.faa.base;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.Collections;
import java.util.Comparator;

public class MyTest2 {

    public static void main(String[] args) {
        ArrayList<Integer> arrayList = new ArrayList<Integer>();
        arrayList.add(-1);
        arrayList.add(3);
        arrayList.add(3);
        arrayList.add(-5);
        arrayList.add(7);
        arrayList.add(4);
        arrayList.add(-9);
        arrayList.add(-7);
        System.out.println("原始数组:");
        System.out.println(arrayList);
// void reverse(List list):反转
        Collections.reverse(arrayList);
        System.out.println("Collections.reverse(arrayList):");
        System.out.println(arrayList);

        Collections.sort(arrayList);

        System.out.println("Collections.sort(arrayList):");
        System.out.println(arrayList);

        //lamda写法
        //Collections.sort(arrayList, (a, b) -> b - a);

        //匿名内部类写法
        Collections.sort(arrayList, new Comparator<Integer>() {
            @Override
            public int compare(Integer o1, Integer o2) {
                return o2 - o1;
            }
        });

        System.out.println("Collections.sort(arrayList) up to low");
        System.out.println(arrayList);

    }


}
```