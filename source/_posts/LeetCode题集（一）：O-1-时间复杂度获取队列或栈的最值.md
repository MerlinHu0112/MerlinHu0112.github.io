---
title: LeetCode题集（一）：O(1)时间复杂度获取队列或栈的最值
comments: true
top: true
date: 2020-03-09 17:14:05
tags:
	- Queue
	- Stack
	- Algorithms
categories:	
	- LeetCode
---

LeetCode刷题过程中遇见“平均时间复杂度O(1)获得队列或栈的最值”系列问题，总结一下。

[面试题59 - II. 队列的最大值](https://leetcode-cn.com/problems/dui-lie-de-zui-da-zhi-lcof/)



<!-- more -->

#### 一、队列的最大值

[面试题59 - II. 队列的最大值](https://leetcode-cn.com/problems/dui-lie-de-zui-da-zhi-lcof/)

1、题目：请定义一个队列并实现函数 max_value 得到队列里的最大值。

 * 要求函数 max_value、push_back 和 pop_front 的均摊时间复杂度都是 `O(1)`。
 * 若队列为空，pop_front 和 max_value 需要返回 -1。

2、图解

队列 `queue` 实现 `MaxQueue` 的基本结构，双端队列 `deque` 保存**最大值序列**（降序）。



代码实现：

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



