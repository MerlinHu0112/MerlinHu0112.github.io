---
title: LeetCode题集（一）：O(1)时间复杂度获取队列或栈的最值
comments: true
top: false
date: 2020-03-09 17:14:05
tags:
	- Algorithms
	- Max Queue
	- Min Stack
categories:	
	- LeetCode
---

LeetCode刷题过程中遇见“平均时间复杂度O(1)获得队列或栈的最值”系列问题，总结一下。

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

![](LeetCode题集（一）：O-1-时间复杂度获取队列或栈的最值/1.gif)

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

（未完待续）