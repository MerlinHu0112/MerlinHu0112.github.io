---
title: HashMap源码阅读笔记
comments: true
top: false
date: 2019-09-05
tags:
	- HashMap
	- Sound Code
categories:
	- 源码学习
---

基于JDK 1.8的HashMap阅读笔记，主要包括如下方面：

- HashMap的属性、构造方法及常用方法源码分析。
- 自定义的类，如何确保Key的一致性？
- HashMap在JDK 1.8与JDK 1.7中的差异。

<!--more-->

#### 1. HashMap的概述

- HashMap允许空的键和空的值，是线程不安全的。Hashtable不允许空的键和空的值，是线程安全的。
- HashMap不保证有序。
- 如果迭代性能很重要，则不要将初始容量设置得过高（或负载因子过低）。
- 多线程结构修改时（添加或删除键值对）必须保证同步，如Collections.synchronizedMap方法返回一个同步的Map。

---

#### 2. 属性

##### 2.1 类的静态属性

（1）**初始容量**：16，且必须为2的幂。

```java
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;
```

> HashMap规定哈希数组的容量为2的次幂。设n为容量。当n为2的幂时，转化成二进制为：100...00。在计算索引时（hash&(n-1)），n-1为：011...11。
>
> 对于小于n-1的哈希值，索引位置就是哈希值；对于大于n-1的哈希值，索引位置就是取模。
>
> 因此，在获取索引时提高了与运算的速度，且保证了散列的均匀性。



（2）**最大容量**：2^30 且应该为2的次幂。

```java
static final int MAXIMUM_CAPACITY = 1 << 30;
```

> 引申：那最大容量为什么不是 `1 << 31` 呢？`1 << 31` 的结果是-2147483648。
>
> 那最大容量为什么不是 `Integer.MAX_VALUE` ，即 `2^31 - 1` 呢？规定哈希数组的容量为2的幂。



（3）**默认负载因子**：选择0.75是空间与时间的均衡。

```java
static final float DEFAULT_LOAD_FACTOR = 0.75f;
```



（4）**链表转树的阈值**：当链表结点个数达到TREEIFY_THRESHOLD时，链表转换为红黑树。

```java
static final int TREEIFY_THRESHOLD = 8;
```

> 引申：为什么选择8？
>
> 理想情况下使用随机的哈希值，节点分布在桶中的频率遵循泊松分布。按照泊松分布的计算可以明确，链表中元素个数达到8时，其概率已经很低。因此，选择8是根据概率计算得到的一个合适的值，它既不会使链表长度不足，又不是令其太长。

（5）

```java
static final int MIN_TREEIFY_CAPACITY = 64;
```

哈希表容量达到MIN_TREEIFY_CAPACITY时，才允许链表转换成红黑树。

- MIN_TREEIFY_CAPACITY不小于TREEIFY_THRESHOLD的4倍。
- **若哈希表容量不足MIN_TREEIFY_CAPACITY，而桶内元素又太多时，直接扩容**。



（6）**树转链表的阈值**：扩容时，原红黑树内结点数量不大于UNTREEIFY_THRESHOLD时，红黑树转换为链表。

```java
static final int UNTREEIFY_THRESHOLD = 6;
```



##### 2.2 成员属性

（1）**modCount**：记录哈希表的结构修改次数。

```java
transient int modCount;
```

使用迭代器时，若调用HashMap的remove()方法试图删除元素时，会抛出ConcurrentModificationException异常（fail-fast策略）。

其底层实现正是基于modCount属性。

每一次对HashMap对象**结构的修改**，modCount加1。在迭代器初始化时，会将该值赋予迭代器的expectedModCount变量。迭代过程中，如调用迭代器的next()方法，会调用checkForComodification()方法以检查modCount是否等于expectedModCount。若是，则正常；若否，则抛出异常。

因此，在使用迭代器时，必须使用迭代器的remove()方法移除集合元素。这一过程的本质是迭代器对象调用集合的remove()方法**并将被修改的modCount值重新赋予expectedModCount变量**，使二者保持一致。

---

#### 3. 节点内部类

（1）Node静态内部类：数组元素和链表节点。

> JDK 1.7中为Entry类

```java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;

    Node(int hash, K key, V value, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }

    public final K getKey()        { return key; }
    public final V getValue()      { return value; }
    public final String toString() { return key + "=" + value; }

    public final int hashCode() {
        return Objects.hashCode(key) ^ Objects.hashCode(value);
    }

    public final V setValue(V newValue) {
        V oldValue = value;
        value = newValue;
        return oldValue;
    }
	// 判断两个Entry是否相等，仅当key和value均相等时，返回true。
    public final boolean equals(Object o) {
        if (o == this)
            return true;
        if (o instanceof Map.Entry) {
            Map.Entry<?,?> e = (Map.Entry<?,?>)o;
            if (Objects.equals(key, e.getKey()) &&
                Objects.equals(value, e.getValue()))
                return true;
        }
        return false;
    }
}
```



（2）TreeNode静态内部类：红黑树的节点。

```java
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
    TreeNode<K,V> parent;  // red-black tree links
    TreeNode<K,V> left;
    TreeNode<K,V> right;
    TreeNode<K,V> prev;    // needed to unlink next upon deletion
    boolean red;
    TreeNode(int hash, K key, V val, Node<K,V> next) {
        super(hash, key, val, next);
    }
    /*其它方法略*/
}
```

---

#### 4. 构造方法

HashMap中的构造方法如下：

```java
public HashMap(int initialCapacity, float loadFactor);
public HashMap(int initialCapacity); // 默认负载因子，0.75
public HashMap(); // 默认的初始容量，16；默认的负载因子，0.75
public HashMap(Map<? extends K, ? extends V> m);
```

（1）使用指定的初始容量和负载因子

```java
public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                           initialCapacity);
    // 初始容量大于最大容量时，将其置为最大容量。
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                           loadFactor);
    this.loadFactor = loadFactor;
    /*
     * 大于initialCapacity且最接近的2的次幂。
     * 随后扩容阈值还是会被重新计算。
     */
    this.threshold = tableSizeFor(initialCapacity);
}
```

threshold：需要扩容时的阈值，在此先对其调用 tableSizeFor()方法

```java
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

此方法返回的是一个**大于形参且最接近的2的幂的值**。**创建哈希表时，threshold阈值会重新赋值**。



（2）其它构造函数

都是调用第一种方法。

```java
public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}
```

```java
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}
```



（3）使用与指定Map相同的映射构造一个新的HashMap

此构造方法使用与指定Map相同的映射构造一个新的HashMap。 使用默认的加载因子（0.75）和足以将映射保存在指定Map中的初始容量创建HashMap。

```java
public HashMap(Map<? extends K, ? extends V> m) {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    putMapEntries(m, false);
}
```

将指定Map中的键值对复制进HashMap中。

```java
final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
    int s = m.size();
    if (s > 0) {
        if (table == null) { // pre-size
            float ft = ((float)s / loadFactor) + 1.0F; // 重新计算初始容量
            int t = ((ft < (float)MAXIMUM_CAPACITY) ?
                     (int)ft : MAXIMUM_CAPACITY);
            if (t > threshold)
                threshold = tableSizeFor(t);
        }
        else if (s > threshold)
            resize();
        for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
            K key = e.getKey();
            V value = e.getValue();
            putVal(hash(key), key, value, false, evict);
        }
    }
}
```

---

#### 5. put方法

HashMap的put方法如下：

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
```

（1）首先看计算key哈希值的方法：

- 当key为null时，哈希值为0；

- 否则，调用Object基类的native的hashCode()方法，且将哈希值的高16位做异或运算。

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

> 引申：为什么要对高16位做异或运算？
>
> putVal方法中的第二个if语句，p = tab[i = (n - 1) & hash] 。我们是根据key的哈希值来保存在散列表中的，我们表默认的初始容量是16，要放到散列表中，就是0-15的位置上。也就是`tab[i = (n - 1) & hash]`。
>
> 可以发现的是：在做`&`运算的时候，仅仅是**后4位有效**。那如果我们key的哈希值高位变化很大，低位变化很小。直接拿过去做`&`运算，这就会导致计算出来的Hash值相同的很多，即哈希冲突严重。使用异或运算，向下传播较高位的影响，，否则由于表范围的限制，这些位将永远不会在索引计算中使用。

*详细解析如下：*

```
假设现有键A和键B，通过调用hashCode方法计算得到的hash分别为:
A: 1111 1111 1111 0000 0000 0000 0000 0001
B: 0000 0000 0000 1111 1111 1111 1111 0001
假设底层数组长度为8，即n=7，二进制表示为：
n: 0000 0000 0000 0000 0000 0000 0000 0111

对A和n做&运算得：
   0000 0000 0000 0000 0000 0000 0000 0001 = 1
对B和n做&运算得：
   0000 0000 0000 0000 0000 0000 0000 0001 = 1
可见，虽然A和B的hash相差甚远，但是它们还是被分入同一个桶中。
```

当有大量的**高位变化明显、但低位不变化**的hash结果时，会造成哈希冲突严重，使得散列不均匀。

为了解决这一问题，采用**用key的哈希值的高16位与其自身做异或运算**的方法。

```
先对A和A的高16位做异或运算，再与n做与运算：
  1111 1111 1111 0000 0000 0000 0000 0001
^ 0000 0000 0000 0000 1111 1111 1111 0000
  1111 1111 1111 0000 1111 1111 1111 0001
& 0000 0000 0000 0000 0000 0000 0000 0111
  0000 0000 0000 0000 0000 0000 0000 0001 = 1

对B和B的高16位做异或或运算，再与n做与运算：
  0000 0000 0000 1111 1111 1111 1111 0001
^ 0000 0000 0000 0000 0000 0000 0000 1111
  0000 0000 0000 1111 1111 1111 1111 1110
& 0000 0000 0000 0000 0000 0000 0000 0111
  0000 0000 0000 0000 0000 0000 0000 0110 = 6
```

经过处理，A和B对应的Node元素将存入数组tab的不同索引！

因此，hash(Object key)方法的目的在于：对于HashMap这种数组总长度一般不大（一般小于2^16）的情形，将哈希值高位的变化传递至低位，使得散列更加均匀。

至于为什么选择 `^` ？这是因为相较于 `&` 和 `|` ，`^` 得到1/0的概率均为50%，更加符合hash算法设计的初衷。



（2）调用putVal方法

- hash：key的哈希值
- key：键
- value：待插入的值
- onlyIfAbsent：true - 不更新已存在的旧值；false - 更新已存在的旧值
- evict：false - 哈希表处于创建模式

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // 散列表不存在或散列表长度为0时，调用resize()方法以初始化哈希表。
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    // 当对应的桶为空时，新建一个Node结点，并将其放入桶中。
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    // 对应的桶不为空，即桶中至少有一个元素。桶中链表的头结点指针此时为p。
    else {
        Node<K,V> e; K k;
        /*
         * 判断数组tab存储的是否为同一个key元素，若是则通过变量e记录之，
         * 以便在下面的代码中用新值value覆盖旧值。
         * 通过分别验证hashCode相等和key相等来保证key的唯一性。
         */
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        // 如果结点p为红黑树结点，则调用数的插入方法插入。
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        /*
         * key不是桶中链表的头结点，则沿链表寻找对应的结点。
         */
        else {
            for (int binCount = 0; ; ++binCount) {
                // 未找到结点，在链表尾部插入新结点
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    /*
                     * 插入新结点后，链表长度达到链转树的阈值时，链表转换为红黑树。
                     * TREEIFY_THRESHOLD-1：减去头结点。
                     */
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                // 找到对应的结点，使用指针p记录该结点。
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        // 使用新值更新旧值，并返回旧值。
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount; // 记录对HashMap结构上的修改次数
    // 哈希表的尺寸超过扩容阈值，执行扩容操作。
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

> 引申：计算哈希数组的下标位置，为何要使用tab[i = (n - 1) & hash]的操作？直接使用哈希值作为数组的索引不行吗？
>
> 答：Object类的hashCode()方法是根据**对象的内存地址经特定算法计算得到的整型参数**，其范围广。HashMap的容量一般不大，显然不合适。
>
> 在HashMap中，与运算是根据其容量大小，**取哈希值一定数量的低位作为下标**，其本质是将哈希值对容量取模。
>
> 以上，包括对哈希值进行异或运算和与运算，其核心目的都是为了使哈希过程随机化和均匀化，避免过度的哈希冲突。



（3）key的一致性

在putVal方法中，判断key是否相同时，使用了如下语句：

```java
p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k)))
```

首先来看Object基类的equals(Object obj)方法，根据 JDK 1.8的API文档，其有如下特性：

- 自反性：对任何非空的 x，x.equals(x) 返回true；
- 对称性：对任何非空的 x 和 y，当且仅当 x.equals(y) 返回 true 时，y.equals(x) 才返回true；
- 传递性：对任何非空的 x、y 和 z，若 x.equals(y) 和 y.equals(z) 均返回true，x.equals(z) 也应该返回true；
- 一致性：对任何非空的 x 和 y，若未加修改，那么多次调用 x.equals(y) 方法，返回结果均保持一致。
- 对任何非空的 x，x.equals(null) 应返回false。
- 对任何非空的 x 和 y，当且仅当 x 和 y 引用相同的对象（即 x==y）时，x.equals(y) 方法才返回true。可见，基类Object的equals方法本质上是**内存地址比较**。

再来看看Object类的hashCode()方法，它有如下约定：

- 对同一个对象多次调用，返回相同的哈希值；
- equals方法返回true的两个对象，应返回相同的哈希值；
- equals方法返回false的两个对象，对返回的哈希值没有明确要求，可相同也可不相同。

综上，关于equals方法和hashCode方法有一个重要的约定：**equals方法返回true时，hashCode方法返回相同的哈希值。**



***那么，对于自定义的 key类，如何保证 key的一致性呢？***

> 结论：重写equals和hashCode方法，以满足上述约定。

*坏例子*

自行定义一个 Key 的类

```java
public class Student {
	private String name;
	private String id;
	
	public Student(String name, String id) {
		this.name = name;
		this.id = id;
	}
	// get/set方法略
}
```

建立 HashMap 对象

```java
public static void main(String[] args) {
    // stA和stB指向的均为同一个人
    Student stA = new Student("张三", "007");
    Student stB = new Student("张三", "007");
    HashMap<Student, Integer> map = new HashMap<>();
    map.put(stA, 97);
    map.put(stB, 75);
    System.out.println(map);
}
```

打印结果为

```java
{Student@15db9742=97, Student@6d06d69c=75}
```

显然，`stA` 和 `stB` 指向同一个对象，二者的 `key` 应该是相同的，但输出结果表明二者的 `key` 是不同的。

在HashMap的源码中，是**通过比较hash相等和key相等来保证key的一致性的**。对于Student类，其equals方法实际上调用的是Object基类的equals方法，即比较 `stA` 和 `stB` 的**内存地址**。

显然，`stA` 和 `stB` 表明同一个人，是指其name和id属性共同确定了张三这个人，但是二者的内存地址是不同的。在这里，我们比较key是否一致，应由name和id属性共同决定！

对于自定义的Key类，应保证**重写equals方法和hashCode方法**，以满足二者的约定条件，保证Key的一致性。

*重写 Student 类的 equals 方法，使其实现比较 name 和 id 属性是否为相同的字符串。*

```java
@Override
public boolean equals(Object obj) {
    if(this==obj) {
        return true;
    }
    if(obj==null) {
        return false;
    }
    if(this.getClass()!=obj.getClass()) {
        return false;
    }
    Student otherObj = (Student) obj;
    if(name==null) {
        if(otherObj.name!=null) {
            return false;
        }
    }else if(!name.equals(otherObj.name)) {
        return false;
    }
    if(id==null) {
        if(otherObj.id!=null) {
            return false;
        }
    }else if(!id.equals(otherObj.id)) {
        return false;
    }
    return true;
}
```

*重写 Student 类的 hashCode 方法，使其符合 equals 方法返回 true 则返回相同哈希值的约定。*

```java
@Override
public int hashCode() {
    final int prime = 31;
    int result = 1;
    result = prime * result + ((name==null) ? 0 : name.hashCode());
    result = prime * result + ((id==null) ? 0 : id.hashCode());
    return result;
}
```

打印结果

```java
{Student@16f482f=75}
```

保证了 Key 的一致性！

---

#### 6. resize方法

final修饰的resize()方法：初始化和扩容时使用。

- 初始化时，将哈希表的初始容量设置为16，扩容阈值设置为12。
- 扩容时，新表的容量和扩容阈值均被设置为旧表的2倍。
- 当旧表容量超过最大容量（2^30）时，无法扩容。只能将扩容阈值设为整型最大值（2^31 - 1）,并返回旧的哈希表。
- JDK 1.8 开始，向链表中插入结点的方式由头插法（JDK 1.7）修改为尾插法。

```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    /***当前旧的哈希表容量大于0***/
    if (oldCap > 0) {
        /*
         * 旧的哈希表容量已超过最大容量（2^30），无法扩容。
         * 只能将扩容阈值设为整型最大值（2^31 - 1）,并返回旧的哈希表。
         */
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // 新表的容量和扩容阈值均被设为旧表的2倍。
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    
    /***当前旧的哈希表容量等于0***/
    // 旧表的扩容阈值大于0，将新表的容量设为该值，新表的扩容阈值设置在第34行。
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    /*
     * 初始化表，表的容量设为默认值，为16；表的扩容阈值默认为12。
     */
    else {               // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    // 设置扩容阈值
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    // 构建新的哈希表
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    
    /***拷贝旧表至新表的操作***/
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                if (e.next == null)
                    // 桶中仅有一个结点时，计算在新表中的索引。
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                    /*
                     * 循环过程实现了将Head和Tail指针分别指向链表的头尾结点。
                     * loHead和loTail表示结点在新表中的索引与旧表一致；
                     * hiHead和hiTail表示结点在新表中的索引为旧表索引+旧表容量
                     */
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        // 原索引
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        // 原索引+旧容量
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    // 新数组中的存储位置：原索引
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    // 新数组中的存储位置：原索引+旧容量
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

扩容后，确定新的哈希数组中的索引的机制：

- 扩容后，哈希表的容量为旧表的2倍。
- 重新依据hashCode & (n-1)计算索引时，哈希值不变、n增大为2倍，**导致最终的计算结果多了一位**。
  - 哈希值对应的位为0，则新增的1对与运算结果无影响，新索引 = 旧索引；
  - 哈希值对应的位为1，则新增的1对与运算结果有影响，新索引 = 旧索引+旧表容量。



**JDK 1.8中重新确定新的哈希数组索引的机制，相较于JDK 1.7有什么异同？**

- 本质：**二者相同**。即JDK 1.8若重新计算hashCode，再进行与操作，得到的结果是不变的，即新索引 = 旧索引或者新索引 = 旧索引+旧表容量。
- 性能：重新计算key的哈希值需要消耗一定的性能，而JDK 1.8只需要执行一次与操作就能得到结果，显然后者更快。



**JDK 1.7形成环状链表的风险？**

JDK 1.7在扩容时，将旧数组上的数据转移至新数组的过程中，其操作为**按旧链表正序遍历，在新的桶中以头插法存入新节点**。这种操作易导致**链表逆序**。

多线程并发扩容时，容易造成环形链表，导致遍历链表时形成死循环（Infinite Loop）。

JDK 1.8采用尾插法，避免上述问题。但需要明确的是，在JDK .18中，**HashMap仍然是线程不安全的类**。

---

#### 7. get方法

get(Object key)方法可返回指定key的值。若指定的key不存在或**显式地为null**，返回null。

```java
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}
```

get方法调用getNode方法，沿链表或红黑树查找与key一致的节点。关于如何保证key的一致性，前已述及。

```java
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        if ((e = first.next) != null) {
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```

------

#### 8. remove方法

```java
public V remove(Object key) {
    Node<K,V> e;
    return (e = removeNode(hash(key), key, null, false, true)) == null ?
        null : e.value;
}
```

调用removeNode()方法

```java
final Node<K,V> removeNode(int hash, Object key, Object value,
                               boolean matchValue, boolean movable) {
    Node<K,V>[] tab; Node<K,V> p; int n, index;
    // 节点在桶中
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (p = tab[index = (n - 1) & hash]) != null) {
        Node<K,V> node = null, e; K k; V v;
        // 链表头节点即是目标节点
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            node = p;
        else if ((e = p.next) != null) {
            // 头节点不是目标节点，首先判断是否已经树化，随后去树中查找
            if (p instanceof TreeNode)
                node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
            else {
                // 尚未树化，遍历链表，锁定目标节点
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key ||
                         (key != null && key.equals(k)))) {
                        node = e;
                        break;
                    }
                    p = e;
                } while ((e = e.next) != null);
            }
        }
        // 移除节点
        if (node != null && (!matchValue || (v = node.value) == value ||
                             (value != null && value.equals(v)))) {
            if (node instanceof TreeNode)
                ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
            else if (node == p)
                tab[index] = node.next;
            else
                p.next = node.next;
            ++modCount; // 修改集合结构的操作，触发modCount值变化
            --size;
            afterNodeRemoval(node);
            return node;
        }
    }
    return null;
}
```

------

#### 9. containsKey方法

调用getNode方法，当key不存在或显式的为null时，皆返回null。

```java
public boolean containsKey(Object key) {
    return getNode(hash(key), key) != null;
}
```

------

#### 10. treeifyBin方法

链表转换为红黑树

```java
final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index; Node<K,V> e;
    /*
     * 只有当哈希表容量不小于MIN_TREEIFY_CAPACITY，链表才能转换为红黑树，
     * 否则，只能扩容处理。
     */
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        resize();
    else if ((e = tab[index = (n - 1) & hash]) != null) {
        TreeNode<K,V> hd = null, tl = null;
        do {
            TreeNode<K,V> p = replacementTreeNode(e, null);
            if (tl == null)
                hd = p;
            else {
                p.prev = tl;
                tl.next = p;
            }
            tl = p;
        } while ((e = e.next) != null);
        if ((tab[index] = hd) != null)
            hd.treeify(tab);
    }
}
```

------

#### 11.  HashMap在JDK 1.7与JDK 1.8中的区别

|      不同点      |        JDK 1.7         |           JDK 1.8           |
| :--------------: | :--------------------: | :-------------------------: |
|    节点实现类    |      Entry<K, V>       | Node<K, V>，TreeNode<K, V>  |
|       结构       |       数组+链表        |      数组+链表+红黑树       |
| 计算key的哈希值  | 4次位运算+5次异或运算  |    1次位运算+1次异或运算    |
| 链表节点插入方式 |         尾插法         |           头插法            |
|    初始化方式    | 单独的inflateTable方法 |     集成在resize方法中      |
| 扩容时计算新索引 |     重新计算（慢）     | 原索引或原索引+旧容量（快） |

---

#### 12. 总结

基于JDK 1.8版本的HashMap。

- HashMap的初始容量默认为16，最大容量为2^30，且容量一定是2的幂。
- 链表转红黑树的条件：哈希表的总节点数大于或等于MIN_TREEIFY_CAPACITY（默认：64），且链表的节点数大于或等于TREEIFY_THRESHOLD（默认：8）。

- HashMap真正初始化哈希表（即初始化哈希数组`table`）是在首次添加键值对，即首次调用put方法时，且真正的初始化方法集成在resize方法中。在JDK 1.7版本中，初始化哈希表是在首次调用put方法，且哈希数组为空的情况下，直接调用inflateTable方法。
- 在计算key的哈希值时，采用高16位异或运算，是为了将高16位的变化引入低16位，使散列更加均匀。
- 对于自定义的key类，应重写equals和hashCode方法，使其满足约定。
- HashMap允许null的键，是因为当key为null时，哈希值为0。因此，HashMap仅允许一个键为null。
- 扩容时，若旧表容量超过最大值，不扩容；成功扩容后，新表容量和阈值均为旧表的2倍。



#### 参考资料

[1] [HashMap源码解析](https://www.jianshu.com/p/d4fee00fe2f8)

[2] [HashMap就是这么简单【源码剖析】](https://www.jianshu.com/p/2c1491d866f7)

[3] [如何将自定义的类对象作为key存储到HashMap中？](https://blog.csdn.net/weixin_41888813/article/details/99715799)