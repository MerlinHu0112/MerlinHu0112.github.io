---
title: 集合源码三：Vector源码阅读笔记
comments: true
top: false
date: 2020-05-06 11:47:24
tags:
	- Vector
categories:
	- 集合源码
---

基于 JDK 1.8 的 Vector 类的源码阅读笔记。

除去 Vector 类是同步、ArrayList 类是非同步，以及二者在扩容机制的微小差异，Vector 类和 ArrayList 是非常相似的。

<!--more-->

#### 1. Vector的概述

Vector 类继承自抽象类 AbstractList，实现了 List、Cloneable、Serializable 和 RandomAccess（标识支持随机访问）接口。

- 底层结构是 Object 类型的数组，可以存放任意元素；
- 支持随机访问，查找操作的时间复杂度为 O(1)。
- 线程安全，这是 Vector 与 ArrayList 的区别之一。
- 迭代器是 fail-fast 的。构造迭代器后，除使用迭代器的 add 或 remove 方法，任何对集合的结构修改均会抛出 ConcurrentModificationException。



#### 2. 字段

##### 2.1 静态常量

（1）最大容量

> 在部分虚拟机上保留 8 个字节。

```java
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
```



##### 2.2 实例变量

（1）数组缓冲区：存放 Vector 数组元素

```java
protected Object[] elementData;
```

（2）容量增量：每次扩容时数组增加的容量。**当 capacityIncrement 小于或等于 0 时，数组双倍扩容，而不是增加 0**。

```java
protected int capacityIncrement;
```

（3）数组大小

```java
protected int elementCount;
```





#### 3. 构造方法

Vector类的构造方法有以下四种：

```java
Vector() // 构造一个Vector实例，初始容量为10，容量增量为0，即扩容时双倍扩容。
Vector(int initialCapacity) // 构造一个Vector实例，指定初始容量，容量增量为0，即扩容时双倍扩容。
Vector(int initialCapacity, int capacityIncrement) // 构造一个Vector实例，指定初始容量和容量增量。
Vector(Collection<? extends E> c) // 构造一个包含集合c所有元素的Vector实例。
```

***Vector 类空参构造函数执行过程***

> Vector 默认的初始容量为 10，容量增量为 0，即执行双倍扩容策略。

（1）首先执行空参构造方法 Vector() 

```java
public Vector() {
    this(10); // 默认初始容量为10
}
```

（2）调用构造方法 Vector(int initialCapacity)

```java
public Vector(int initialCapacity) {
    this(initialCapacity, 0); // 初始容量为10，容量增量为0。
}
```

（3）调用构造方法 Vector(int initialCapacity, int capacityIncrement)

```java
public Vector(int initialCapacity, int capacityIncrement) {
    super(); // 调用父类的空参构造方法，将父类的modCount字段初始为0。
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal Capacity: "+
                                           initialCapacity);
    this.elementData = new Object[initialCapacity];
    this.capacityIncrement = capacityIncrement;
}
```



#### 4. 常用方法

```java
public boolean add(E e) // 在尾部添加元素
public void add(int index, E element) // 在指定位置插入元素
public E remove(int index) // 删除指定位置元素，返回被删除的元素
public boolean remove(Object o) // 删除首次出现的元素o
public E set(int index, E element) // 使用新元素e覆盖指定位置的元素，并返回旧元素
public E get(int index) // 返回指定位置的元素
public boolean contains(Object o) // 查找指定的元素是否存在
public Object[] toArray() // 转换为数组
public Enumeration<E> elements() // 返回枚举对象
```

##### （1）add(E e) 方法

> 同步的 add 方法内部需要调用一系列辅助**非同步方法**，好处在于避免大量的额外同步开销。

在尾部添加元素 `e` 。Vector 类的 `add(E e)` 方法与 `addElement(E obj)` 功能相同，但前者返回值类型为 `boolean` 、后者无返回值。 

```java
public synchronized boolean add(E e) {
    modCount++;
    ensureCapacityHelper(elementCount + 1);
    elementData[elementCount++] = e;
    return true;
}
```

*调用非同步的 ensureCapacityHelper 方法，确保内部容量能满足添加新元素*

```java
// Vector类的同步方法调用此方法，在避免额外的同步开销的前提下，检查底层数组容量是否足够。
private void ensureCapacityHelper(int minCapacity) {
    // overflow-conscious code
    if (minCapacity - elementData.length > 0)
        grow(minCapacity); // 容量不足，扩容。
}
```

*容量不足，调用 grow 方法扩容*

```java
private void grow(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;
    // 当容量增量不大于0时，双倍扩容；否则按容量增量增加。
    int newCapacity = oldCapacity + ((capacityIncrement > 0) ?
                                     capacityIncrement : oldCapacity);
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity; // 扩容后的容量仍不足，直接更新为所需的最小容量。
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity); // 若新容量超过最大值
    elementData = Arrays.copyOf(elementData, newCapacity); // 拷贝
}
```

*若新容量超过最大值，调用 hugeCapacity 方法*

```java
private static int hugeCapacity(int minCapacity) {
    // 若允许的最小容量发生溢出、变为负数，抛出异常。
    if (minCapacity < 0) // overflow
        throw new OutOfMemoryError();
    // 若允许的最小容量大于数组容量限制，返回int整数最大值，即Integer.MAX_VALUE；
    // 否则返回Integer.MAX_VALUE - 8。
    return (minCapacity > MAX_ARRAY_SIZE) ?
        Integer.MAX_VALUE :
    MAX_ARRAY_SIZE;
}
```

------

##### （2）add(int index, E element) 方法

此方法是非同步，但它内部调用同步的方法。

```java
public void add(int index, E element) {
    insertElementAt(element, index);
}
```

*调用同步方法 insertElementAt，插入元素*

```java
public synchronized void insertElementAt(E obj, int index) {
    modCount++;
    if (index > elementCount) {
        throw new ArrayIndexOutOfBoundsException(index
                                                 + " > " + elementCount);
    }
    ensureCapacityHelper(elementCount + 1); // 检查容量
    // 移动元素，空出index索引处。
    System.arraycopy(elementData, index, elementData, index + 1, elementCount - index);
    elementData[index] = obj;
    elementCount++; // 数组大小加1
}
```

------

##### （3）remove(int index) 方法

> 与 ArrayList 类相似，remove 方法仅检查索引是否越过右端边界，若成立则抛出异常。**此方法不检查索引是否为负数，如果索引为负数则在访问数组时抛出 ArrayIndexOutOfBoundsException。**
>
> 不同的是，ArrayList 中抛出两个不同的异常（IndexOutOfBoundsException + ArrayIndexOutOfBoundsException），而 Vector 中均抛出 ArrayIndexOutOfBoundsException。

```java
public synchronized E remove(int index) {
    modCount++;
    if (index >= elementCount) // 仅判断数组右端是否越界
        throw new ArrayIndexOutOfBoundsException(index);
    E oldValue = elementData(index); // elementData方法返回指定元素

    int numMoved = elementCount - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    elementData[--elementCount] = null; // Let gc do its work

    return oldValue; // 返回被删除的元素
}
```

------

##### （4）remove(Object o) 方法

注意，当指定元素 `o` 为 `null` 且 Vector 集合中存在 `null` 元素，返回 `true` 。

> 在 Vector 类的 remove 方法中，removeElement 和 removeElementAt 方法分别执行一次 `modCount++`，即 remove 方法中 `modCount++` 操作被执行两次。

```java
public boolean remove(Object o) {
    return removeElement(o);
}
```

*方法内部调用同步的 removeElement 方法：*

```java
public synchronized boolean removeElement(Object obj) {
    modCount++;
    int i = indexOf(obj); // 经多层调用，底层调用同步的indexOf方法，获取指定元素索引。
    if (i >= 0) { // i不为-1，即指定元素存在。
        removeElementAt(i); // 删除元素
        return true;
    }
    return false;
}
```

*调用同步的 indexOf 方法，查找指定元素*

```java
public synchronized int indexOf(Object o, int index) {
    if (o == null) { // 对象o为null时
        for (int i = index ; i < elementCount ; i++)
            if (elementData[i]==null)
                return i;
    } else { // 对象o不为null时
        for (int i = index ; i < elementCount ; i++)
            if (o.equals(elementData[i]))
                return i;
    }
    return -1; // 指定对象o不存在，返回-1。
}
```

*被 remove 方法调用，删除指定索引位置的元素*

```java
public synchronized void removeElementAt(int index) {
    modCount++;
    // 判断索引的合法性
    if (index >= elementCount) {
        throw new ArrayIndexOutOfBoundsException(index + " >= " +
                                                 elementCount);
    }
    else if (index < 0) {
        throw new ArrayIndexOutOfBoundsException(index);
    }
    int j = elementCount - index - 1;
    if (j > 0) {
        // 移动待删除元素后的全部元素
        System.arraycopy(elementData, index + 1, elementData, index, j);
    }
    elementCount--;
    elementData[elementCount] = null; /* to let gc do its work */
}
```

------

##### （5）set(int index, E element) 方法

> setElementAt(E obj, int index) 方法仅更新索引 index 处的元素、不返回旧元素。

```java
public synchronized E set(int index, E element) {
    if (index >= elementCount) // 检查索引合法性
        throw new ArrayIndexOutOfBoundsException(index);

    E oldValue = elementData(index); // 获取旧元素
    elementData[index] = element; // 新元素覆盖
    return oldValue; // 返回旧元素
}
```

------

##### （6）get(int index) 方法

```java
public synchronized E get(int index) {
    if (index >= elementCount)
        throw new ArrayIndexOutOfBoundsException(index);

    return elementData(index); // 获取并返回索引index处的元素
}
```

------

##### （7）contains(Object o) 方法

```java
public boolean contains(Object o) {
    return indexOf(o, 0) >= 0; // 调用同步的indexOf方法
}
```

------

##### （8）toArray() 方法

```java
public synchronized Object[] toArray() {
    return Arrays.copyOf(elementData, elementCount);
}
```

*调用工具类 Arrays 的静态方法 copyOf*

```java
// 拷贝原数组，副本长度指定为newLength，以null截断或填充副本。
// 当newLength大于原数组的长度时，以null填充副本中多余的部分。
public static <T> T[] copyOf(T[] original, int newLength) {
    return (T[]) copyOf(original, newLength, original.getClass());
}
```

*继续调用工具类 Arrays 的静态方法*

```java
public static <T,U> T[] copyOf(U[] original, int newLength, 
                               Class<? extends T[]> newType) {
    @SuppressWarnings("unchecked")
    // 获取副本数组的类型
    T[] copy = ((Object)newType == (Object)Object[].class)
        ? (T[]) new Object[newLength]
        : (T[]) Array.newInstance(newType.getComponentType(), newLength);
    // 调用native方法完成复制，其中待复制长度为原数组长度和指定值中的最小值。
    // original.length == newLength，完全相同；
    // original.length > newLength，发生截断；
    // original.length < newLength，以null填充副本中多余的位置。
    System.arraycopy(original, 0, copy, 0,
                     Math.min(original.length, newLength));
    return copy;
}
```

------

##### （9）elements() 方法

> 此方法返回枚举类型对象。ArrayList 类不提供此方法。

```java
public Enumeration<E> elements() {
    return new Enumeration<E>() {
        int count = 0;

        public boolean hasMoreElements() {
            return count < elementCount;
        }

        public E nextElement() {
            synchronized (Vector.this) {
                if (count < elementCount) {
                    return elementData(count++);
                }
            }
            throw new NoSuchElementException("Vector Enumeration");
        }
    };
}
```



#### 5. 总结

Vector 类与 ArrayList 类的比较如下表：

|              |                  Vector                   |                          ArrayList                           |
| :----------: | :---------------------------------------: | :----------------------------------------------------------: |
| 是否线程安全 |        线程不一定安全（参考第6节）        |                          线程不安全                          |
|   扩容机制   |           按增量增加 或 2倍扩容           |                          1.5倍扩容                           |
| 空参构造方法 | 默认初始容量为10、2倍扩容，构造时分配内存 | 默认初始容量为10，构造空数组，首次添加元素扩容至默认容量、完成内存分配 |
|    迭代器    |   Iterator + ListIterator + Enumeration   |                   Iterator + ListIterator                    |

> 注：详细扩容过程参考 ArrayList 源码解析。



#### 6. 问题

1、对 Vector 类的操作一定是线程安全的吗？

 答：Vector 类通过 synchronized 对外提供线程安全的操作方法，但对 Vector 容器的**复合操作**不一定是线程安全的！

*示例*

```java
import java.util.Vector;

public class VectorThreadUnsafe {

    public static void main(String[] args) {
        Vector<Integer> vector = new Vector<>();
        vector.add(1);

        Thread aThread = new Thread(new Runnable() {
            @Override
            public void run() {
                if(!vector.isEmpty()){
                    try {
                        Thread.sleep(100);
                    }catch (Exception e){
                        e.printStackTrace();
                    }
                    vector.remove(0);
                    System.out.println("线程aThread的remove方法被执行");
                }
            }
        });

        Thread bThread = new Thread(new Runnable() {
            @Override
            public void run() {
                if(!vector.isEmpty()){
                    vector.remove(0);
                    System.out.println("线程bThread的remove方法被执行");
                }
            }
        });
        
        aThread.start();
        bThread.start();
    }
}
```

运行结果如下：

```java
线程bThread的remove方法被执行
Exception in thread "Thread-0" java.lang.ArrayIndexOutOfBoundsException: Array index out of range: 0
	at java.util.Vector.remove(Vector.java:834)
	at cn.merlin.test.concurrency.VectorThreadUnsafe$1.run(VectorThreadUnsafe.java:20)
	at java.lang.Thread.run(Thread.java:748)
```

根据报错信息可知，线程 `aThread` 在执行 remove 方法时出现异常！

**具体细节：**线程 `aThread` 执行判空操作后让出执行权，线程 `bThread` 执行判空操作且删除了 `vector` 实例中的唯一一个元素。随后线程 `aThread` 恢复执行，试图删除已经被线程 `bThread` 删除的元素，故抛出异常！

***额外加锁以保证复合操作的安全***

```java
import java.util.Vector;

public class VectorThreadSafe {

    public static void main(String[] args) {
        Vector<Integer> vector = new Vector<>();
        vector.add(1);

        Thread aThread = new Thread(new Runnable() {
            @Override
            public void run() {
                synchronized (vector){
                    if(!vector.isEmpty()){
                        try {
                            Thread.sleep(100);
                        }catch (Exception e){
                            e.printStackTrace();
                        }
                        vector.remove(0);
                        System.out.println("线程aThread的remove方法被执行");
                    }
                }
            }
        });

        Thread bThread = new Thread(new Runnable() {
            @Override
            public void run() {
                if(!vector.isEmpty()){
                    vector.remove(0);
                    System.out.println("线程bThread的remove方法被执行");
                }
            }
        });

        aThread.start();
        bThread.start();
    }
}
```

**总结：** Vector 类是同步的，仅意味着其对外提供的简单操作是线程安全的。但这些操作复合后，多个线程交叉访问容器可能导致并发异常！



#### 参考资料

[1] [Vector并非是绝对的线程安全类](https://blog.csdn.net/u011676300/article/details/81102259)

[2] [Vector 是线程安全的？](https://blog.csdn.net/xdonx/article/details/9465489)