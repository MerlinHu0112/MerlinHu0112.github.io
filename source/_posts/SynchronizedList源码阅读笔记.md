---
title: 集合源码四：SynchronizedList源码阅读笔记
comments: true
top: false
date: 2020-05-13 12:00:22
tags:
	- SynchronizedList
	- Collections	
categories:
	- 集合源码
---

JDK 除提供 Vector、ConcurrentHashMap 等同步容器外，通过工具类 **Collections** 的静态工具方法可获取一些同步的集合，如 synchronizedList、synchronizedMap、synchronizedSet 等。

本文主要内容有：

- 分析 Collections 类的内部类 SynchronizedList 的源码。
- 对同步的集合对象的复合操作不一定是线程安全的。
- 比较 Vector 类与 SynchronizedList 类。

<!--more-->

#### 1. SynchronizedList 源码

> 本小节涉及的 SynchronizedCollection、SynchronizedList、SynchronizedRandomAccessList，均为 Collections 类的内部类。

工具类 Collections 中常见的获取同步的集合的方法如下：

```java
public static <T> Collection<T> synchronizedCollection(Collection<T> c)
public static <T> List<T> synchronizedList(List<T> list)
public static <K,V> Map<K,V> synchronizedMap(Map<K,V> m)
public static <T> Set<T> synchronizedSet(Set<T> s)
```

以 synchronizedList 为例，通过调用静态方法可获取同步的 List 。

```java
List list = Collections.synchronizedList(new LinkedList<>());
```



##### 1.1 初始化过程

*静态方法 synchronizedList 的源码如下*

```java
public static <T> List<T> synchronizedList(List<T> list) {
    return (list instanceof RandomAccess ? // 判读list是否支持随机访问（LinkedList不支持）
            new SynchronizedRandomAccessList<>(list) :
            new SynchronizedList<>(list)); // 调用SynchronizedList类的构造方法
}
```

*继续看静态内部类 SynchronizedList，其继承 SynchronizedCollection 类、实现 List 接口*

```java
static class SynchronizedList<E> extends SynchronizedCollection<E> 
								 implements List<E> {
    
    private static final long serialVersionUID = -7754090372962971524L;

    final List<E> list;
	
    SynchronizedList(List<E> list) {
        super(list); // 调用父类的构造方法
        this.list = list;
    }
}
```

*继续看静态内部类 SynchronizedCollection，其实现了 Collection 和 Serializable 接口*

```java
static class SynchronizedCollection<E> implements Collection<E>, Serializable {
    
    private static final long serialVersionUID = 3053995032091335093L;

    final Collection<E> c;
    final Object mutex; // 锁对象

    SynchronizedCollection(Collection<E> c) {
        this.c = Objects.requireNonNull(c); // 调用requireNonull方法，判断集合c是否为null
        mutex = this; // 指定锁对象为当前对象
    }
}
```

至此，通过工具类 Collections 的静态方法 synchronizedList 获取到了 SynchronizedList 对象，即同步的 List 对象。



##### 1.2 同步方法

以静态内部类 SynchronizedList 为例，通过 synchronized 为方法的调用者加锁，实现同步。

```java
// add(E e)方法继承自SynchronizedCollection类
public boolean add(E e) {
    synchronized (mutex) { // 锁对象为方法的调用者
        return c.add(e);
    }
}

// 删除
public E remove(int index) {
    synchronized (mutex) {return list.remove(index);}
}

// 修改
public E set(int index, E element) {
    synchronized (mutex) {return list.set(index, element);}
}

// 查找
public E get(int index) {
    synchronized (mutex) {return list.get(index);}
}
```



##### 1.3 获取迭代器的方法

特别地，静态内部类 SynchronizedCollection、SynchronizedList、SynchronizedMap 及 SynchronizedSet 中的 **获取迭代器的方法是未同步的**。

```java
public ListIterator<E> listIterator() {
    return list.listIterator(); // 必须手动加锁
}
```

*使用时需要手动为其加锁*

```java
List list = Collections.synchronizedList(new ArrayList<>());
synchronized (list){
    Iterator itr = list.iterator();
    while (itr.hasNext()){
        System.out.println(itr.next());
    }
}
```



#### 2. 线程安全性

在 Vector 类和 Collections 的内部类 SynchronizedCollection、SynchronizedList 中，均是通过**关键字 synchronized** 对类提供的方法（获取迭代器除外）进行加锁，以实现同步。

与 Vector 类相似，SynchronizedCollection、SynchronizedList 等对外提供的、synchronized 修饰的方法是线程安全的，**但多线程复合操作容器不一定是线程安全的！**具体分析见 [Vector 源码笔记](https://merlinhu0112.github.io/2020/05/06/Vector%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB%E7%AC%94%E8%AE%B0/)。



#### 3. SynchronizedList 与 Vector

Vector 是 java.util 包中的一个类， SynchronizedList 是 java.util.Collections 中的一个静态内部类。

- Vector 内部使用同步方法，SynchronizedList 内部使用同步代码块。
- SynchronizedList 具有很好的扩展性，可以将底层结构是链表的 LinkedList 类转换成线程安全的类，而Vector 的底层结构固定是数组。
- Vector 类获取迭代器的方法是同步的，而 SynchronizedList 类获取迭代器的方法未实现同步，在使用时需要自行加锁。
- SynchronizedList 可以使用默认的锁对象（集合本身），也可以在构造方法中指定锁对象。Vector 类的锁对象只能是 Vector 实例。



#### 4. 总结

对于工具类 Collections 中获取同步的集合对象的静态方法，其本质是在内部维护一个锁对象（mutex），在方法（除获取迭代器的方法）中使用同步代码块以实现同步。锁对象为方法的调用者。



#### 参考资料

[1] [Java 中 Collections.synchronizedList(List list) 原理分析](https://blog.csdn.net/weixin_38575051/article/details/94000044)

[2] [Java集合之Vector源码分析](https://www.cnblogs.com/hujingnb/p/10181577.html)