---
title: 集合源码二：ArrayList源码阅读笔记
comments: true
top: false
date: 2020-05-06 11:43:12
tags:
	- ArrayList
categories:
	- 集合源码
---

基于 JDK 1.8 的 ArrayList 类的源码阅读笔记，重点如下：

- ArrayList 的底层结构及扩容机制。
- 如何安全地对集合的结构进行修改。
- 迭代器的 remove 方法的本质。
- Iterator 和 ListIterator 两种迭代器的区别。

<!-- more -->

#### 1. ArrayList的概述

ArrayList 类继承自抽象类 AbstractList，实现了 List、Cloneable、Serializable 和 RandomAccess（标识支持随机访问）接口。

- 底层结构是 Object 类型的数组，可以存放任意元素；
- 支持随机访问，查找操作的时间复杂度为 O(1)。
- 未实现同步，可通过 Collections 类的 synchronizedList 方法获得线程安全的类。
- 迭代器是 fail-fast 的。构造迭代器后，除使用迭代器的 add 或 remove 方法，任何对集合的结构修改均会抛出 ConcurrentModificationException。



#### 2. 字段

##### 2.1 静态常量

（1）默认初始容量

```java
private static final int DEFAULT_CAPACITY = 10;
```

（2）共享空数组实例之一：用于空的 ArrayList 实例

```java
private static final Object[] EMPTY_ELEMENTDATA = {};
```

（3）共享空数组实例之二：用于默认大小的空的 ArrayList 实例（看空参构造函数）

```java
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
```

（4）数组最大容量

> 在部分虚拟机上保留 8 个字节，但 **ArrayList 数组的最大容量可取到 Integer.MAX_VALUE** 。

```java
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
```



##### 2.2 实例变量

（1）数组缓冲区：存放 ArrayList 数组元素

```java
transient Object[] elementData; 
```

对于 `DEFAULTCAPACITY_EMPTY_ELEMENTDATA` 实例，添加第一个元素时，数组自动扩充为 `DEFAULT_CAPACITY` 大小。



#### 3. 构造方法

ArrayList类的构造方法有以下三种：

```java
ArrayList() // 构造一个初始容量为10的空列表。
ArrayList(int initialCapacity) // 构造一个指定初始容量的空列表。
ArrayList(Collection<? extends E> c) // 构造一个包含指定集合元素的列表，其顺序由集合的迭代器返回。
```

（1）ArrayList()

> **首次添加元素时，完成内存分配**。由空数组扩容至默认大小，即 10。

```java
public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}
```

（2）ArrayList(int initialCapacity)

```java
public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {
        this.elementData = new Object[initialCapacity]; // 新建Object数组，内存分配。
    } else if (initialCapacity == 0) {
        this.elementData = EMPTY_ELEMENTDATA; // 空数组
    } else {
        throw new IllegalArgumentException("Illegal Capacity: "+
                                           initialCapacity);
    }
}
```

（3）ArrayList(Collection<? extends E> c)

```java
public ArrayList(Collection<? extends E> c) {
    elementData = c.toArray();
    // 判断传入的集合c是否为空
    if ((size = elementData.length) != 0) {
        // 判断引用的数组类型。若不是Object类型数组，将其转换。
        if (elementData.getClass() != Object[].class)
            elementData = Arrays.copyOf(elementData, size, Object[].class);
    } else {
        // 传入的集合c为空
        this.elementData = EMPTY_ELEMENTDATA;
    }
}
```



#### 4. 常用方法

```java
public boolean add(E e) // 在数组尾部添加元素
public void add(int index, E element) // 在指定位置插入元素
public boolean addAll(Collection<? extends E> c) // 添加指定集合c的全部元素
public boolean addAll(int index, Collection<? extends E> c) // 从List指定位置开始，插入集合c全部元素。
public E remove(int index) // 移除指定位置元素
public boolean remove(Object o) // 删除首次出现的元素o
public E set(int index, E element) // 使用新元素e覆盖指定位置的元素，并返回旧元素。
public E get(int index) // 返回指定位置的元素
public boolean contains(Object o) // 查找指定的元素是否存在
public Object[] toArray() // 将列表转换为数组
public Iterator<E> iterator() // 返回迭代器对象，只能向后遍历
public ListIterator<E> listIterator() // 返回迭代器对象，可双向访问
```

##### （1）add(E e) 方法

在数组尾部添加元素 `e` 。

```java
public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // 确保内部容量
    elementData[size++] = e; // 存入元素值e
    return true;
}
```

*调用 ensureCapacityInternal 方法，确保内部容量能满足添加新元素*

```java
private void ensureCapacityInternal(int minCapacity) {
    ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
}
```

*调用 calculateCapacity 方法，计算本次操作所需要的最小容量*

```java
private static int calculateCapacity(Object[] elementData, int minCapacity) {
    // 判断数组是否是用默认构造函数初始化的
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        // 若实例由空参函数构造，在首次添加元素时，数组扩容至默认大小，即10。
        return Math.max(DEFAULT_CAPACITY, minCapacity); // Math.max(10, 1);
    }
    return minCapacity;
}
```

*调用 ensureExplicitCapacity 方法，检查旧数组是否需要扩容*

```java
private void ensureExplicitCapacity(int minCapacity) {
    modCount++;

    // overflow-conscious code
    if (minCapacity - elementData.length > 0)
        grow(minCapacity); // 扩容
}
```

*旧数组容量不足，调用 grow 方法扩容*

```java
private void grow(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1); // （近似）1.5倍扩容机制
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity; // 1.5倍扩容后的容量仍不足，直接更新为所需的最小容量。
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity); // 若新数组容量超过最大值
    // minCapacity is usually close to size, so this is a win:
    elementData = Arrays.copyOf(elementData, newCapacity); // 拷贝数组
}
```

*若新数组容量超过最大值，调用 hugeCapacity 方法*

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

```java
public void add(int index, E element) {
    rangeCheckForAdd(index); // 检查指定索引是否合法

    ensureCapacityInternal(size + 1); // 检查数组容量是否足够
    System.arraycopy(elementData, index, elementData, index + 1,
                     size - index); // 从旧数组的index索引至结束，拷贝至新数组。
    elementData[index] = element; // 插入指定元素
    size++;
}
```

*向指定位置插入元素时，首先调用 rangeCheckForAdd 方法，检查指定索引是否合法*

```java
private void rangeCheckForAdd(int index) {
    if (index > size || index < 0) // 判断索引是否越界
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}
```

------

##### （3）addAll(Collection<? extends E> c) 方法

```java
public boolean addAll(Collection<? extends E> c) {
    Object[] a = c.toArray();
    int numNew = a.length;
    ensureCapacityInternal(size + numNew);  // 检查数组容量
    System.arraycopy(a, 0, elementData, size, numNew);
    size += numNew;
    return numNew != 0;
}
```

------

##### （4）addAll(int index, Collection<? extends E> c) 方法

```java
public boolean addAll(int index, Collection<? extends E> c) {
    rangeCheckForAdd(index); // 首先判断index是否合法。index是List的索引，而不是集合c。

    Object[] a = c.toArray();
    int numNew = a.length;
    ensureCapacityInternal(size + numNew);  // Increments modCount

    int numMoved = size - index;
    if (numMoved > 0)
        System.arraycopy(elementData, index, elementData, index + numNew,
                         numMoved); // 移动原数组中的部分元素，空出位置以插入集合c中的元素。

    System.arraycopy(a, 0, elementData, index, numNew); // 插入集合c中的元素
    size += numNew;
    return numNew != 0;
}
```

------

##### （5）remove(int index) 方法

> rangeCheck 方法仅检查索引是否越过右端边界，若成立则抛出 IndexOutOfBoundsException。**此方法不检查索引是否为负数，如果索引为负数则在访问数组时抛出 ArrayIndexOutOfBoundsException。**

```java
public E remove(int index) {
    rangeCheck(index); // 检查索引合法性

    modCount++; // 删除操作也要更新modCount
    E oldValue = elementData(index); // 获取指定位置的元素

    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved); // 移动数组，覆盖被删除元素的位置
    elementData[--size] = null; // clear to let GC do its work

    return oldValue;
}
```

*检查索引是否越过右端边界，即索引是否大于最大索引值*

```java
private void rangeCheck(int index) {
    if (index >= size)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}
```

------

##### （6）remove(Object o) 方法

注意，当指定元素 `o` 为 `null` 且 ArrayList 集合中存在 `null` 元素，返回 `true` 。

```java
public boolean remove(Object o) {
    if (o == null) {
        for (int index = 0; index < size; index++)
            if (elementData[index] == null) {
                fastRemove(index);
                return true;
            }
    } else {
        for (int index = 0; index < size; index++)
            if (o.equals(elementData[index])) {
                fastRemove(index);
                return true;
            }
    }
    return false;
}
```

*方法内部调用 fastRemove 方法：*

- 不检查边界
- 不返回被删除元素

```java
private void fastRemove(int index) {
    modCount++;
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    elementData[--size] = null; // clear to let GC do its work
}
```

------

##### （7）set(int index, E element) 方法

```java
public E set(int index, E element) {
    rangeCheck(index); // 检查边界

    E oldValue = elementData(index); // 获取旧元素
    elementData[index] = element; // 新元素覆盖
    return oldValue; // 返回旧元素
}
```

------

##### （8）get(int index) 方法

```java
public E get(int index) {
    rangeCheck(index); // 检查边界
    return elementData(index); // 返回指定位置元素
}
```

------

##### （9）contains(Object o) 方法

```java
public boolean contains(Object o) {
    return indexOf(o) >= 0;
}
```

*调用 indexOf 方法，返回指定元素的首次出现的索引*

```java
// 返回指定元素在此列表中首次出现的索引；如果此列表不包含该元素，则返回-1。
public int indexOf(Object o) {
    if (o == null) {
        for (int i = 0; i < size; i++)
            if (elementData[i]==null)
                return i;
    } else {
        for (int i = 0; i < size; i++)
            if (o.equals(elementData[i]))
                return i;
    }
    return -1;
}
```

------

##### （10）toArray() 方法

```java
public Object[] toArray() {
    return Arrays.copyOf(elementData, size); // 拷贝
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

##### （11）iterator() 方法

```java
public Iterator<E> iterator() {
    return new Itr(); // Itr是ArrayList的内部类，对它的介绍见第5小节。
}
```

------

##### （12）listIterator() 方法

```java
public ListIterator<E> listIterator() {
    return new ListItr(0); // ListItr是ArrayList的内部类，对它的介绍见第6小节
}
```



#### 5. 内部类Itr

内部类Itr是一种优化后的迭代器，它只支持向后访问列表。

```java
private class Itr implements Iterator<E> {
    int cursor;       // 游标
    int lastRet = -1; // 上次返回的元素的索引
    int expectedModCount = modCount;

    Itr() {}

    public boolean hasNext() {
        return cursor != size;
    }

	......    
}
```

------

##### （1）迭代器的next方法

```java
@SuppressWarnings("unchecked")
public E next() {
    checkForComodification(); // 检查是否在迭代过程中使用非迭代器的方法对集合进行结构化修改
    int i = cursor;
    if (i >= size)
        throw new NoSuchElementException(); // 数组越界，不存在下一元素。
    // ArrayList.this.elementData表示ArrayList中的Object数组
    Object[] elementData = ArrayList.this.elementData;
    if (i >= elementData.length) // 再一次判断是否越界，防止其它线程修改了集合结构。
        throw new ConcurrentModificationException();
    cursor = i + 1;
    return (E) elementData[lastRet = i]; // 使用lastRet标识此次返回的元素
}
```

*调用 checkForComodification 方法，检查是否在迭代过程中使用非迭代器的方法对集合进行结构化修改*

```java
// 简单地判断迭代器的expectedModCount参数与集合的modCount参数是否相同
final void checkForComodification() {
    if (modCount != expectedModCount) 
        throw new ConcurrentModificationException();
}
```

------

##### （2）remove() 方法

删除上一次执行 next 方法的元素。

> 迭代过程中，使用迭代器的 remove 方法才能安全地移除集合元素。

```java
public void remove() {
    // 判断是否执行过next方法
    if (lastRet < 0)
        throw new IllegalStateException();
    checkForComodification(); // 检查是否发生结构化修改的异常

    try {
        ArrayList.this.remove(lastRet); // 调用集合的remove方法
        cursor = lastRet; // 集合的remove方法删除元素后，返回原被删除元素的索引
        lastRet = -1; // 重置lastRet
        expectedModCount = modCount; // 修改迭代器的expectedModCount参数
    } catch (IndexOutOfBoundsException ex) {
        throw new ConcurrentModificationException();
    }
}
```



#### 6. 内部类ListItr

ListItr 也是一种内部迭代器类，它与 Itr 的区别在于，ListItr 可以**向前**和**向后**遍历列表，而 Itr 只能向后遍历。

```java
// 继承Itr类，实现ListIterator接口。
private class ListItr extends Itr implements ListIterator<E> {
	
    // 判断是否有前驱元素
    public boolean hasPrevious() {
        return cursor != 0;
    }
	
    // 返回前驱元素索引
    public int previousIndex() {
        return cursor - 1;
    }
	
    // 返回前驱元素
    @SuppressWarnings("unchecked")
    public E previous() {
        checkForComodification();
        int i = cursor - 1;
        if (i < 0)
            throw new NoSuchElementException();
        Object[] elementData = ArrayList.this.elementData;
        if (i >= elementData.length)
            throw new ConcurrentModificationException();
        cursor = i;
        return (E) elementData[lastRet = i];
    }

    ......
}
```



#### 7. 总结

- ArrayList 的底层结构是数组，当数组长度不足时，通过扩容（1.5 倍）机制扩容，扩容的本质是复制数组。
- ArrayList 常用的构造函数是空参构造函数。空参构造函数得到的是**空数组**，首次添加元素时**扩容至默认值，即 10 。**
- **扩容机制：** grow(int minCapacity) 传入最小容量参数，首先执行 1.5 倍扩容方法，当扩容后的容量仍不满足最小容量时，将其直接置为最小容量。随后对得到的新容量进行校验，若其大于 Integer.MAX_VALUE - 8，执行 hugeCapacity(int minCapacity) 方法。在这一方法中，若最小容量发生溢出，直接抛出异常；否则判断最小容量是否大于 Integer.MAX_VALUE - 8。若是，将新容量更新为 Integer.MAX_VALUE；否则新容量为 Integer.MAX_VALUE - 8。
- ArrayList 的源码中，规定最大容量是 **Integer.MAX-VALUE - 8**，这是因为部分虚拟机上会保留 8 个字节。但是根据 **hugeCapacity** 方法可发现，实际上最大可取 **Integer.MAX_VALUE**。在这种情况下，部分虚拟机（保留 8 个字节的）会报 OOM。
- ArrayList 继承父类的 AbstractList 中的 modCount 字段，记录集合发生结构化修改（**删除、添加元素，更新已存在的元素不是结构化修改**）的次数。迭代器中维护 expectedModCount 字段，在初始化迭代器时，将集合的 modCount 赋予迭代器的 expectedModCount。在执行迭代器的方法时，首先判断二者是否相等，否的话抛出 ConcurrentModificationException 异常。
- 迭代器的 remove 方法的本质是**调用集合的 remove 方法并更新迭代器的 expectedModCount 的值。**
- 迭代器 Iterator 和 ListIterator 都可以遍历集合，只是后者可以额外地向前遍历，其实现是将当前游标减1。
- 迭代器的 next 方法，首先调用 checkForComodification 方法检查是否对集合进行了修改，在获取元素值前判断索引是否越界，防止其它线程修改集合结构。