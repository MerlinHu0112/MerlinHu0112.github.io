---
title: LeetCode题集（一）：O(1)时间复杂度的操作
comments: true
top: false
date: 2020-03-09 17:14:05
tags:
	- Algorithms
	- Max Queue
	- Min Stack
	- LRU
categories:	
	- LeetCode
---

LeetCode刷题过程中遇见“O(1)时间复杂度的操作”系列问题，总结一下。

<!-- more -->

#### 一、最大队列

[面试题59 - II. 队列的最大值](https://leetcode-cn.com/problems/dui-lie-de-zui-da-zhi-lcof/)

##### 1. 题目

请定义一个队列并实现函数 max_value 得到队列里的最大值。

 * 要求函数 max_value、push_back 和 pop_front 的均摊时间复杂度都是 `O(1)`。
 * 若队列为空，pop_front 和 max_value 需要返回 -1。

##### 2. 图解

队列 `queue` 实现 `MaxQueue` 的基本结构，双端队列 `deque` 保存**最大值序列**（降序）。

当元素 `ele` 入队时，分别进入 `queue` 和 `deque`。

- 入 `queue` 时，元素 `ele` 直接入队。
- 入 `deque` 时，先从队尾开始寻找所有比`ele` 小的元素，直至遇见首个大于 `ele` 的元素或已经到了队头。随后从队尾开始，移除所有小于 `ele` 的元素并将 `ele` 入队。

（图源：[如何解决 O(1) 复杂度的 API 设计题](https://leetcode-cn.com/problems/dui-lie-de-zui-da-zhi-lcof/solution/ru-he-jie-jue-o1-fu-za-du-de-api-she-ji-ti-by-z1m/)）

![](LeetCode题集（一）：O(1)时间复杂度的操作/1.gif)

##### 3. 代码实现：

```java
public class MaxQueue {
	
	// 队列queue，实现MaxQueue的基本结构
	private Queue<Integer> queue;
	// 双端队列（可两端出、入队），用于记录最大值递减序列
	private Deque<Integer> deque;
	
	public MaxQueue() {
         queue = new LinkedList<Integer>();
         deque = new ArrayDeque<Integer>();
    }
    
    public int max_value() {
    	return deque.size() > 0 ? deque.peek() : -1;
    }
    
    public void push_back(int value) {
    	queue.add(value);
    	while(deque.size() > 0 && deque.peekLast() < value ) {
    		// 移除双端队列后部小于value的元素，直至遇见一个
    		// 大于value的元素或双端队列为空时
    		deque.pollLast();
    	}
    	deque.add(value); // 双端队列队尾入队
    }
    
    public int pop_front() {
    	// 获取队首元素，若队列为空则为-1
    	int headEle = queue.size() > 0 ? queue.poll() : -1;
    	if(deque.size() > 0 && deque.peek() == headEle) {
    		deque.poll();
    	}
    	return headEle;
    }
}
```



---



#### 二、最小栈

[面试题 03.02. 栈的最小值](https://leetcode-cn.com/problems/min-stack-lcci/)

##### 1. 题目

请设计一个栈，除了常规栈支持的pop与push函数以外，还支持min函数，该函数返回栈元素中的最小值。执行push、pop和min操作的时间复杂度必须为 `O(1)` 。

##### 2. 思路

类似最大队列的实现。

利用辅助栈保存最小值集合，当前待压栈元素若不小于辅助栈栈顶元素，将该元素压入辅助栈中；否则，辅助栈不变。

##### 3. 代码实现

```java
class MinStack {
    
    private Stack<Integer> stack; // 存放元素
    private Stack<Integer> auxStack; // 辅助栈，存放最小值序列

    public MinStack() {
        stack = new Stack<>();
        auxStack  = new Stack<>();
    }
    
    public void push(int x) {
        stack.push(x);
        // 辅助栈不为空且栈顶元素小于x时，x不进入辅助栈
        if(!auxStack.empty() && auxStack.peek() < x ){
            return;
        }
        // 否则，x被压入辅助栈
        auxStack.push(x);
    }
    
    public void pop() {
        if(stack.empty()){
            return;
        }
        // 最小栈栈顶元素等于辅助栈栈顶元素时，说明当前最小值将被弹栈，
        // 因此辅助栈栈顶元素同时弹栈
        int temp = stack.peek();
        if(auxStack.peek()==temp){
            auxStack.pop();
        }
        // 最小栈栈顶元素弹栈
        stack.pop();
    }
    
    public int top() {
        if(stack.empty()){
            return -1;
        }
        return stack.peek();
    }
    
    public int getMin() {
        if(!auxStack.empty()){
            return auxStack.peek();
        }
        return -1;
    }
}
```



---



#### 三、LRU缓存

[面试题 16.25. LRU缓存](https://leetcode-cn.com/problems/lru-cache-lcci/)

**LRU缓存**：Least Recently Used，即最近最久未使用的，是一种常见的缓存淘汰算法。当存储的数据总量达到缓存限制时，删除最久未使用过的数据。

##### 1. 题目

设计和构建一个“最近最少使用”缓存，该缓存会删除最近最少使用的项目。缓存应该从键映射到值（允许你插入和检索特定键对应的值），并在初始化时指定最大容量。

当缓存被填满时，它应该删除最近最少使用的项目。

它应该支持以下操作： 获取数据 get 和 写入数据 put 。

获取数据 get(key) - 如果密钥 (key) 存在于缓存中，则获取密钥的值（总是正数），否则返回 -1。
写入数据 put(key, value) - 如果密钥不存在，则写入其数据值。当缓存容量达到上限时，它应该在写入新数据之前删除最近最少使用的数据值，从而为新的数据值留出空间。



##### 2. 思路

为实现常数时间内 get 和 put 键值对，采用**哈希表**存储，如 HashMap。

在操作键值对的同时，需要额外维护使用键值对的先后顺序，且维护也应在常数时间内完成。**双向链表**是一个不错的选择。

双向链表维护访问先后顺序。其头结点（或尾结点）表示最近最久未使用过的数据。（以尾结点表示最近最久未使用为例）

- 插入新的键值对时，将 `key` 存入双向链表的头结点，表示最近被使用的数据。
- 当更新键值对时，将 `key` 从双向链表相应位置移向头结点。
- 当缓存已满时，移除双向链表的尾结点并移除散列表中对应的键值对即可。

上述三种操作，第一种和第三种均很容易在常数时间内实现，而第二种需要额外的辅助才能实现O(1)的时间复杂度，因为双向链表查找的时间复杂度为O(N)。

Java提供的 `java.util.LinkedHashMap` 基于 `HashMap` 和双链表很容易实现LRU缓存。



##### 3. 基于LinkedHashMap（有序字典）的实现

Java提供的 LinkedHashMap 类默认按照键值对插入顺序维护双向链表，同时也提供一个特殊的构造函数，以实现按访问顺序维护双向链表。

- initialCapacity：初始化容量
- loadFactor：加载因子
- accessOrder：true 表示按访问顺序，false 表示按插入顺序

```java
public LinkedHashMap(int initialCapacity,
                         float loadFactor,
                         boolean accessOrder) {
    super(initialCapacity, loadFactor);
    this.accessOrder = accessOrder;
}
```

由于 LinkedHashMap 类提供的 removeEldestEntry 方法默认返回 false，因此需要重写该方法，并指定删除最久未使用数据的规则（如当数据量超过缓存容量时）。

```java
protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
    return false;
}
```

*代码实现*

```java
class LRUCache {

	private LinkedHashMap<Integer, Integer> map;
	private int MAX_CACHE_SIZE; // 缓存容量

    public LRUCache(int capacity) {
    	MAX_CACHE_SIZE = capacity;
        // 初始容量根据缓存容量和加载因子计算得到，不应设置为等于缓存容量。
        // 若初始容量被设置为缓存容量，易导致rehash操作发生，降低算法性能。
    	map = new LinkedHashMap<>((int) Math.ceil(capacity / 0.75f) + 1, 0.75f, true){
    		@Override
    		protected boolean removeEldestEntry(Map.Entry eldest) {
    			return size() > MAX_CACHE_SIZE;
    		}
    	};
    }
    
    public int get(int key) {
    	Integer value = map.get(key);
    	return value == null ? -1 : value;
    }
    
    public void put(int key, int value) {
    	map.put(key, value);
    }
}
```



##### 4、基于哈希表和双向链表的实现

如思路部分所述，可使用双向链表保存访问键值对的顺序信息。

双向链表插入操作的时间复杂度为O(1)，但查询操作的时间复杂度为O(N)。若被访问的键值对对应的链表结点位于链表中部，那么查找操作将时总时间复杂度提升为O(N)。那如何解决这一问题呢？

参照 LinkedHashMap 的源码实现。将键值对保存在链表结点中，哈希表保存 **key 和 结点内存地址**，这样，定位至相应结点的时间复杂度就被优化为O(1)。

*结点结构如下*

```java
private class Node{
    int key;
    int value;
    Node prev;
    Node next;
}
```

*哈希表结构如下*

```java
private HashMap<Integer, Node> cache;
```

（图源：[LRU 缓存机制](https://leetcode-cn.com/problems/lru-cache/solution/lru-huan-cun-ji-zhi-by-leetcode/)）

![](LeetCode题集（一）：O(1)时间复杂度的操作/2.jpg)



头尾指针为辅助指针。



*算法实现*

```java
class LRUCache {
	
	// 双向链表的结点
	private class Node{
		int key;
		int value;
		Node prev;
		Node next;
	}
	private Node head; // 辅助的头指针
	private Node tail; // 辅助的尾指针
	
	// 在头部插入结点
	private void insertHeadNode(Node node) {
		node.prev = head;
		node.next = head.next;
		head.next.prev = node;
		head.next = node;
	}
	
	// 移除尾结点
	private Node removeTailNode() {
		Node target = tail.prev;
		removeNode(target);
		// 返回被移除的结点指针，是为了再移除尾结点后，
		// 从哈希表中删除对应的记录。
		return target;
	}
	
	// 移除结点
	private void removeNode(Node node) {
		Node pNode = node.prev;
		Node qNode = node.next;
		pNode.next = qNode;
		qNode.prev = pNode;
	}
	
	// 移动结点至头结点位置
	private void moveToHead(Node node) {
		// 先移除，再插入
		removeNode(node);
		insertHeadNode(node);
	}
	
	private HashMap<Integer, Node> cache;
	private final float DEFAULT_LOAD_FACTOR = 0.75f; // 加载因子
	private int MAX_CACHE_SIZE; // 缓存容量;
	
    public LRUCache(int capacity) {
    	int initialCapacity = (int) Math.ceil(capacity / DEFAULT_LOAD_FACTOR) + 1; // 哈希表初始容量
    	cache = new HashMap<>(initialCapacity, DEFAULT_LOAD_FACTOR);
    	MAX_CACHE_SIZE = capacity;
    	// 初始化头尾指针
    	head = new Node();
    	tail = new Node();
    	head.next = tail;
    	tail.prev = head;
    }
    
    public int get(int key) {
    	Node node = cache.get(key);
    	if(node==null) {
    		return -1;
    	}
    	moveToHead(node); // 更新访问记录
    	return node.value;
    }
    
    public void put(int key, int value) {
    	Node node = null;
    	// 键已存在
    	if(cache.containsKey(key)) {
    		node = cache.get(key);
    		node.value = value;
    		moveToHead(node);
    		return;
    	}
    	// 键不存在
    	node = new Node();
    	node.key = key;
    	node.value = value;
    	insertHeadNode(node); // 插入头结点
    	cache.put(key, node);
    	// 还需判断哈希表容量是否已超过缓存容量
    	if(cache.size() > MAX_CACHE_SIZE) {
    		Node target = removeTailNode(); // 移除尾结点
    		cache.remove(target.key);
    	}
    	return;
    }
}
```


