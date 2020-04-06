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

相关题目：[第53题-最大子序和](https://leetcode-cn.com/problems/maximum-subarray/ )。

`Kadane`算法求解最大子序和问题的分析见[动态规划和分治法求解最大子序和问题](https://merlinhu0112.github.io/2020/02/25/%E5%8A%A8%E6%80%81%E8%A7%84%E5%88%92%E5%92%8C%E5%88%86%E6%B2%BB%E6%B3%95%E6%B1%82%E8%A7%A3%E6%9C%80%E5%A4%A7%E5%AD%90%E5%BA%8F%E5%92%8C%E9%97%AE%E9%A2%98/)。

##### 1. 题目

题目：求最大子序和。给定一个整数数组 `nums` ，找到一个具有最大和的连续子数组（子数组最少包含一个元素），返回其最大和。

用例：`[-2, 1, -3, 4, -1, 2, 1, -5, 4]`

正确输出结果：`6`

##### 2. 实现

```java
int maxSubArray(int[] nums) {

    int currSum = nums[0];
    int maxSum = nums[0];

    for(int i=1; i<nums.length; i++) {
        currSum = Math.max((currSum+nums[i]), nums[i]);
        maxSum = Math.max(maxSum, currSum);
    }
    return maxSum;
}
```

##### 3. 性能

- 时间复杂度：O(N)
- 空间复杂度：O(1)

---

#### 最长上升子序列

相关题目：[第300题-最长上升子序列](https://leetcode-cn.com/problems/longest-increasing-subsequence/)

##### 1. 题目

题目：给定一个无序的整数数组，找到其中最长上升子序列的长度。

用例：`[10, 9, 2, 5, 3, 7, 101, 18]`

输出：`4`

##### 2. 分析

用数组 `dp` 保存以数组 `nums` 下标 `0` 为起点、下标 `i` 为终点的局部区间最长上升子序列的长度，将其称为 *局部最长上升子序列长度*。

逐个更新 `dp` 数组。易知，计算 `dp[i]` 的值，若满足：

> 0 <= j < i，且 nums[j] < nums[i]

即对于局部区间 `[0, j]`，元素 `nums[i]` 是满足上升条件的。

 状态转移方程为：

> dp[i] = max{ dp[j] } + 1， 0 <= j < i，且 nums[j] < nums[i]

##### 3. 实现

```java
int lengthOfLIS(int[] nums) {
    if(nums==null || nums.length==0){
        return 0;
    }
    
    int[] dp = new int[nums.length]; // 保存不同索引位置处对应的局部最大上升子序列长度
    Arrays.fill(dp, 1); // 局部最长上升子序列长默认为1，即假设完全不存在上升序列
    int maxLength = 1; // 全局最长上升子序列长度

    // 双循环，逐个计算索引i位置处的最大上升子序列长度，最后求出全局最大值
    for(int i=1; i<dp.length; i++){
        for(int j=0; j<i; j++){
            // 只有当nums[i]大于nums[j]时，严格满足上升条件
            if(nums[i] > nums[j]){
                dp[i] = Math.max(dp[i], dp[j]+1);
            }
        }
        maxLength = Math.max(maxLength, dp[i]);
    }
    return maxLength;
}
```

*对内循环中的赋值语句的理解*

```java
dp[i] = Math.max(dp[i], dp[j]+1);
```

若写成下式是**错误的**。

```java
dp[i] = dp[j]+1;
```

因为，索引 `j` 对应的局部最长上升子序列长度值，不一定是区间 `[0, i-1]` 中的最大值！

##### 4. 性能

- 时间复杂度：O(N^2) 
- 空间复杂度：O(N)

---

#### 整数拆分

相关题目：[第343题-整数拆分](https://leetcode-cn.com/problems/integer-break/)，[面试题14-I.剪绳子](https://leetcode-cn.com/problems/jian-sheng-zi-lcof/)。

##### 1. 题目

题目：给定一个正整数 `n`，将其拆分为**至少**两个正整数的和，并使这些整数的乘积最大化。 返回可以获得的最大乘积。

条件：2 <= n <= 58 ，且 2 <= m <= n，其中 `m` 是划分的个数。

##### 2. 分析

设数组 `dp` 中的任意元素 dp[i] 保存整数 `i` 对应的最大乘积，其中 i >= 2。

边界条件为：

> dp[2] = 1

其中 dp[0] 和 dp[1] 为无效参数，仅起到填充作用，以使 `dp` 数组的索引 `i` 与正整数 `n` 对应。

对于整数 `i` ，可将其划分为 `j` 和 `i-j` ，其中 1<= j < i。

`j * (i-j)` 正是整数 `i` 拆分、求积的一种形式，但这不一定是最大值。因为对于由 `i` 划分得到的 `j` ，它对应的 `dp[j]` 不一定等于 `j`，即对整数 `j` 进一步划分，或许还能得到更大的乘积。

综上，应在 j * ( i - j ) 和 dp[j] * ( i - j ) 中选择较大者。

##### 3. 实现

```java
int integerBreak(int n) {
    int[] dp = new int[n+1];
    dp[2] = 1; // 边界条件
    for(int i=3; i<=n; i++){
        for(int j=1; j<i; j++){
            // 状态转移方程
            dp[i] = Math.max(Math.max((j*(i-j)), (dp[j]*(i-j))), dp[i]);
        }
    }
    return dp[n];
}
```

##### 4. 性能

- 时间复杂度：O(N^2) 
- 空间复杂度：O(N)


