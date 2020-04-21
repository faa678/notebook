## <center>HashMap</center>



HashMap = 数组 + 线性链表 + 红黑树

* 数组：指定下标查找快（根据key的映射值）
* 线性链表：查找到在数组中的映射值位置后，在链表上新增或删除数据快
* 红黑树：链表过长的话，查找慢。红黑树查找、插入、删除快（O(logn)）



#### HashMap的13个成员变量：

1. 默认初始容量：16，必须是2的幂 

```java
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;//aka 16，数组长度
```

2. 最大容量：2^30

```java
static final int MAXIMUM_CAPACITY = 1 << 30;
```

3. 默认装载因子：0.75，综合时间和空间考虑得到的，泊松分布，0.75的话碰撞最小

```java
static final float DEFAULT_LOAD_FACTOR = 0.75;
```

4. 实际存储Key，Value的数组，被封装成Node

```java
transient Node<K, V>[] table;
```

5. 键值对数

```java
transient int size;
```

6. 当前HashMap修改的次数，这个变量用来保证fail-fast机制

```java
transient int modCount;
```

7. 阈值(数组大小)，下次需要扩容时的值，等于 容量 * 加载因子

```java
int threshold;
```

8. 桶的树化阈值(链表大小)，当桶中链表长度达到8时可能会转化成红黑树

```java
static final int TREEIFY_THRESHOLD = 8;
```

9. 桶的链表还原阈值(树大小)，当桶中红黑树大小变成6时转化成链表

```java
static final int UNTREEIFY_THRESHOLD = 6;
```

10. 在转变成树前，还要判断键值对数量（桶数量）是否大于64，只有大于才会发生转化。

```java
static final int MIN_TREEIFY_CAPACITY = 64;
```

11. 缓存的键值对集合

```java
transient Set；
```



#### HashMap结构：数组 + 链表 + 红黑树

![avatar](/home/faa/pictures/论文/HashMap.png)



#### 源码解析

构造函数 <font color=purple>`HashMap(int initialCapacity, float loadFactor)`</font>：

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
    //threshold是扩容阈值，threshold = capacity * loadFactor，此处threshold = capacity，因为threshold在put方法中进行了重新计算
    this.threshold = tableSizeFor(initialCapacity);//tableSizeFor(int cap)保证HashMap总是用2的幂作为哈希表的大小
}
```

解析：  

			* <font color=purple>`MAXIMUM_CAPACITY`</font> = 1 << 30，大于<font color=purple>`MAXIMUM_CAPACITY`</font>的初始值，重置为<font color=purple>`MAXIMUM_CAPACITY`</font>
			* <font color = purple>`threshold`</font>是扩容阈值，threshold = capacity * loadFactor，此处threshold = capacity，因为threshold在put方法中进行了重新计算
			* <font color=purple>`tableSizeFor(int cap)`</font>保证了HashMap总是用大于等于设定大小的2的幂作为哈希表的（数组）大小，源码如下：

<font color=purple>`tableSizeFor(int cap)`</font>函数：

```java
static final int tableSizeFor(int cap) {
    //令找到的目标值大于或等于原值。如二进制1000，十进制8。如果不对它减1，将得到10000，即16
    int n = cap - 1;
    //让最高位的1后面的位全变1
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    //保证n是2的幂
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

解析：

* 为了找到key的位置在哈希表的哪个槽，需要计算hash(key) % length，但是 % 比  & 慢很多，所以用 & 代替 %，为了保证&的计算结果等于%的结果，需要把length - 1，也就是hash(key) & (length - 1), 也就是说hash % length == hash & (length - 1)的前提是length是2的n次幂

* length为偶数时，length - 1为奇数，保证了hash & (len - 1)的最后一位可能是0，也可能是1，可以保证散列的均匀。

* 如果length为奇数，只会被散列到偶数下标位置，浪费空间。
* 因此，length取2的整数次幂，为了使不同的hash值发生碰撞的概率较小，均匀散列，并提高效率



移位操作示例：

​	n = 1 << 30时：

```java
01 00000 00000 00000 00000 00000 00000 (n)   
01 10000 00000 00000 00000 00000 00000 (n |= n >>> 1)    
01 11100 00000 00000 00000 00000 00000 (n |= n >>> 2)    
01 11111 11000 00000 00000 00000 00000 (n |= n >>> 4)    
01 11111 11111 11111 00000 00000 00000 (n |= n >>> 8)    
01 11111 11111 11111 11111 11111 11111 (n |= n >>> 16)    
```



构造函数<font color=purple>`HashMap(Map<? extends K, ? extends V> m)`</font>：

```java
//构造一个和制定Map有相同mappings的HashMap，初始容量为DEFAULT_LOAD_FACTOR = 0.75
public HashMap(Map<? extends K, ? extends V> m) {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    putMapEntries(m, false);
}
```



主要函数<font color=purple>`putMapEntries(Map<? extends K, ? extends V> m, boolean evict)`</font>：

```java
final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
    int s = m.size();//模板map的大小
    if (s > 0) {
        //table是节点数组
        //如果table未初始化，先初始化一些变量，table的初始化是在put时
        if (table == null) { // pre-size
            //根据模板map的size计算要创建的HashMap的容量
            float ft = ((float)s / loadFactor) + 1.0F;
            int t = ((ft < (float)MAXIMUM_CAPACITY) ?
                     (int)ft : MAXIMUM_CAPACITY);
            //要创建的HashMap的容量
            if (t > threshold)
                threshold = tableSizeFor(t);
        }
        //如果table已经初始化了并且模板map大小 > table的threshold，使用resize()扩容
        else if (s > threshold)
            resize();
        //遍历待插入的map，将每一个<Key, Value>插入本HashMap中
        for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
            K key = e.getKey();
            V value = e.getValue();
            //调用的是putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict)进行的元素插入
            putVal(hash(key), key, value, false, evict);
        }
    }
}
```



HashMap的<font color=purple>`put(K key, V value)`</font>函数：

```java
public V put(K key, V value) {
    //也调用了putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict)进行的元素插入
    return putVal(hash(key), key, value, false, true);
}
```



扰动函数<font color=purple>`hash(Object key)`</font>方法：

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

解析：

​	让key的hash值的高16位也参与路由运算；key为null时，放在数组的第0个。

	* h是hashcode。h >>> 16用来取出h的高16位，如下：

```java
0000 0100 1011 0011  1101 1111 1110 0001 
>>> 16 
0000 0000 0000 0000  0000 0100 1011 0011
```

* (h = key.hashCode()) ^ (h >>> 16)

  jdk1.7中有indexFor(int h, int length)方法。<font color=red>jdk1.8</font>使用<font color=red>tab[(n - 1) & hash]</font>，但原理没有变

```java
//返回数组下标
static int indexFor(int h, int length) {
	return h & (length - 1);
}
```

​		大多数情况下length小于2 ^ 16，所以return h & (length - 1)的结果始终是h的低位与(length - 1)进行&运算。如果让哈希值地位更加随机，那么&结果就更加随机。如何让哈希值的低位更加随机，就是让其与高位异或。

* 用 ^ 不用 & 和 | 

  &和|都会使结果偏向0或1，不是均匀的概念，所以用^。



<font color=purple>`putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict)`</font>函数：

```java
/**
* Implements Map.put and related methods.
*
* @param hash hash for key
* @param key the key
* @param value the value to put
* @param onlyIfAbsent if true, don't change existing value：为true时，插入已存在key值取消插入，默认false
* @param evict if false, the table is in creation mode.
* @return previous value, or null if none
*/
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    //tab: 引用当前HashMap的散列表
    //p: 当前散列表的元素
    //n: 散列表数组的长度
    //i: 路由寻址结果
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    
    //延迟初始化，第一次调用putVal时会初始化HashMap对象中最好费内存的散列表。
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;//resize()返回Node数组
    
    //找到的桶位为null，将当前（key，value）封装一个Node放入当前桶位中
    if ((p = tab[i = (n - 1) & hash]) == null)//(n - 1) & hash是路由算法，获取
        tab[i] = newNode(hash, key, value, null);
    
    //找到的桶位不为null
    else {
        //e: node临时元素; k: 表示一个临时key
        Node<K,V> e; K k;
        //在桶位中找到了与当前要插入的key一致的元素，后续进行替换操作
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        //红黑树的情况
        else if (p instanceof TreeNode)
            //执行红黑树的插入算法
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        //链表的情况
        else {
            for (int binCount = 0; ; ++binCount) {
                //找到链尾，说明没找到，将当前Node插入
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    //binCount >= 7时（因为binCount从0开始），表示已经有8个元素了，再加元素就树化
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);//树化算法
                    break;
                }
                //e表示p.next，上个if中赋值的
                //如果找到，break，e还是null
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        //替换操作
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    //表示散列表结构被修改的次数
    ++modCount;
    //插入新元素，size(元素个数)自增，如果自增后大于扩容阈值(数组长度 * 加载因子)，扩容
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

解析：

​	HashMap使用链表法避免哈希冲突，链表长度大于TREEFY_THRESHOLD（默认为8）时，将链表转为红黑树；小于UNTREEFY_THRESHOLD（默认为6）时，又会转回链表。



扩容resize()函数：

​	<font color=red>为了解决哈希冲突导致的链化影响查询效率的问题，扩容可以缓解该问题。</font>

```java
final Node<K,V>[] resize() {
    //扩容前的哈希表
    Node<K,V>[] oldTab = table;
    //扩容前的table数组长度
    int oldCap = (oldTab == null) ? 0 : oldTab.length;//new出来的哈希表没有放数据的时候为null，延迟初始化
    //扩容之前的扩容阈值，也就是触发本次扩容的阈值
    int oldThr = threshold;
    //扩容后的table数组长度和扩容阈值
    int newCap, newThr = 0;
    
    //条件如果成立，说明HashMap中的散列表已经初始化过了，是一次正常扩容
    if (oldCap > 0) {
        //扩容之前的table数组大小已经达到最大值后，则不扩容
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;//将阈值改成int的最大值
            return oldTab;
        }
        //如果扩容后的table数组大小 < 最大值并且扩容前数组大小 >= 最小值
        //这种情况下，下一次扩容的阈值等于当前阈值翻倍
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    
    //oldCap == 0, oldThr > 0
    //1. new HashMap(initCap, loadFactor);
    //2. new HashMap(initCap);
    //3. new HashMap(map); 并且map有数据
    //oldThr都被赋为了tableSizeFor(),是扩容后的capacity，而不是最终的threshold，延迟初始化
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;//2的幂
    
    //oldCap == 0, oldThr == 0
    //new HashMap()
    else {               // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    
    //newThr为0时，通过newCap和loadFactor计算出一个newThr
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
   
    //真正的threshold
    threshold = newThr;
    
    //创建出一个更大的数组
    @SuppressWarnings({"rawtypes","unchecked"})
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    //说明HashMap本次扩容之前，table不为null
    if (oldTab != null) {
        //遍历当前table
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                //方便GC
                oldTab[j] = null;
                
                //当前桶位只有一个元素，从未发生过碰撞，直接计算出当前元素在新数组中的位置
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                
                //当前节点已经树化，进行红黑树的rehash
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                
                //桶位已经形成链表，进行链表的rehash
                else { // preserve order
                    //低位链表：存放在扩容之后的数组的下表位置，与当前数组下表位置一致
                    Node<K,V> loHead = null, loTail = null;
                    //高位链表：存放在扩容之后的数组的下表位置为: 当前数组下标位置 + 扩容之前数组的长度
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        //hash-> ... 1 1111
                        //hash->.... 0 1111
                        //oldCap 0b 10000
                        //最高位 == 0, 这是索引不变的链表
                        if ((e.hash & oldCap) == 0) {//求下标的时候用的是cap - 1
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        //最高位 == 1, 这是索引发生改变的链表
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```





