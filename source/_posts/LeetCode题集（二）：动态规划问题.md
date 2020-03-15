---
title: LeetCode题集（二）：动态规划问题
comments: true
top: false
date: 2020-03-15 22:15:15
tags:
	- Algorithms
	- DP
categories:
	- LeetCode
---

总结在LeetCode刷题过程中遇见的动态规划求解的问题。

有关 DP 的详细介绍见[五大常用算法之动态规划](https://merlinhu0112.github.io/2020/01/02/%E4%BA%94%E5%A4%A7%E5%B8%B8%E7%94%A8%E7%AE%97%E6%B3%95%E4%B9%8B%E5%8A%A8%E6%80%81%E8%A7%84%E5%88%92/)

<!--more-->

#### 最大子序和问题

相关题目：LeetCode[第53题-最大子序和](https://leetcode-cn.com/problems/maximum-subarray/ )。

`Kadane`算法求解最大子序和问题的分析见[动态规划和分治法求解最大子序和问题](https://merlinhu0112.github.io/2020/02/25/%E5%8A%A8%E6%80%81%E8%A7%84%E5%88%92%E5%92%8C%E5%88%86%E6%B2%BB%E6%B3%95%E6%B1%82%E8%A7%A3%E6%9C%80%E5%A4%A7%E5%AD%90%E5%BA%8F%E5%92%8C%E9%97%AE%E9%A2%98/)。

##### 1. 题目

题目：求最大子序和。给定一个整数数组 `nums` ，找到一个具有最大和的连续子数组（子数组最少包含一个元素），返回其最大和。

用例：[-2, 1, -3, 4, -1, 2, 1, -5, 4]

正确输出结果：6



##### 2. 实现

```java
public int maxSubArray(int[] nums) {

    int currSum = nums[0];
    int maxSum = nums[0];

    for(int i=1; i<nums.length; i++) {
        currSum = Math.max((currSum+nums[i]), nums[i]);
        maxSum = Math.max(maxSum, currSum);
    }
    return maxSum;
}
```

算法总结：

- 时间复杂度：O(n)
- 空间复杂度：O(1)

---

#### 整数拆分

相关题目：LeetCode[第343题-整数拆分](https://leetcode-cn.com/problems/integer-break/)，[面试题14-I.剪绳子](https://leetcode-cn.com/problems/jian-sheng-zi-lcof/)。

##### 1. 题目

此类型题大致题意是：将整数 `n` 拆分成 `m` 份，求最大积。

##### 2. 分析

