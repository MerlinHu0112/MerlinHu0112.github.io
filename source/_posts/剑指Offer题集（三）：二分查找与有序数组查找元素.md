---
title: 剑指Offer题集（三）：二分查找与有序数组查找元素
comments: true
top: false
date: 2020-04-06 16:25:33
tags:
	- Binary Search
	- Sorted Array
categories:
	- 剑指Offer
---

在无序的数组中查找目标值是否存在，只能通过遍历数组的方式，其时间复杂度为O(N)。

在**有序**的数组中查找目标值是否存在，可以通过遍历数组的方式，时间复杂度为O(N)。考虑到有序这一特性，还可以通过**二分查找**，使时间复杂度降至**O(log N)**。

<!--more-->

##### 面试题11. 