---
title: 剑指Offer题集（四）：矩阵搜索问题
comments: true
top: false
date: 2020-04-09 13:00:41
tags:
	- DFS
	- BFS
	- Backtracking
categories:
	- 剑指Offer
---

矩阵搜索问题通常可通过深度优先搜索（DFS）或广度优先搜索（BFS）解决。本文总结了在刷题过程中遇见的可采用 **DFS + 回溯** 解决的矩阵搜索问题。

核心：由当前位置向指定方向搜索（递归过程），若搜索结果不满足要求，则回溯至原位置。

<!--more-->

#### 面试题12. 矩阵中的路径

##### 1. 题目

请设计一个函数，用来判断在一个矩阵中是否存在一条包含某字符串所有字符的路径。路径可以从矩阵中的任意一格开始，每一步可以在矩阵中向左、右、上、下移动一格。如果一条路径经过了矩阵的某一格，那么该路径不能再次进入该格子。

[力扣-《剑指Offer》-面试题12. 矩阵中的路径](https://leetcode-cn.com/problems/ju-zhen-zhong-de-lu-jing-lcof/)

[力扣-79. 单词搜索](https://leetcode-cn.com/problems/word-search/)

##### 2. 思路

DFS+回溯：类似于[八皇后](https://merlinhu0112.github.io/2020/01/13/%E4%BA%94%E5%A4%A7%E5%B8%B8%E7%94%A8%E7%AE%97%E6%B3%95%E4%B9%8B%E5%9B%9E%E6%BA%AF%E7%AE%97%E6%B3%95/)问题，由二维数组的每一个位置开始搜索。若当前位置字符符合要求，则**递归搜索**当前位置的**上下左右**位置，观察其是否依然符合。在搜索过程中，对于已访问过的位置，需要将其**特殊标记**。若发生回溯，则在回溯过程中取消已标记的位置。

##### 3. 实现

```java
class Solution {
    public boolean exist(char[][] board, String word) {
        char[] arr = word.toCharArray();
        for(int i=0; i<board.length; i++){
            for(int j=0; j<board[0].length; j++){
                if(helper(board, arr, i, j, 0)){
                    return true;
                }
            }
        }
        return false;
    }

    /**
     * 回溯法
     * @param k 字符串word的索引
     * @return
     */
    private static boolean helper(char[][] board, char[] arr,
                                  int i, int j, int k){
        // 索引越界或字符不匹配，直接返回false
        if(i<0 || i>board.length-1 || j<0 || j>board[0].length-1
                || arr[k]!=board[i][j]){
            return false;
        }
        // 以完成全部字符串的匹配，返回true
        if(k==arr.length-1){
            return true;
        }
        
        char temp = board[i][j];
        board[i][j] = '+'; // 占位，标识此位置字符已经被访问过
        // 若索引为k的字符是匹配的，则继续匹配索引为k+1的字符
        // 左,右,上,下
        boolean flag = helper(board, arr, i, j-1, k+1) || helper(board, arr, i, j+1, k+1)
                || helper(board, arr, i-1, j, k+1) || helper(board, arr, i+1, j, k+1);
        board[i][j] = temp; // 还原
        return flag;
    }
}
```

---

#### 面试题13. 机器人的运动范围

##### 1. 题目

地上有一个 `m` 行 `n` 列的方格，从坐标 `[0,0]` 到坐标 `[m-1,n-1]` 。一个机器人从坐标 `[0, 0] `的格子开始移动，它每次可以向左、右、上、下移动一格（不能移动到方格外），也不能进入**行坐标和列坐标的数位之和大于k的格子**。请问该机器人最多能够到达多少个格子？

[力扣-《剑指Offer》-面试题13. 机器人的运动范围](https://leetcode-cn.com/problems/ji-qi-ren-de-yun-dong-fan-wei-lcof/)

##### 2. 思路

DFS+回溯：类似于第12题的搜索过程。机器人从当前位置出发，探索相邻的位置。特别地，若当前位置是符合要求的，那么当前位置的**左邻位**和**上邻位**一定是符合要求的！因此在本题中，机器人的搜索策略可以简化为：若当前位置可行，仅**向右**和**向下**搜索。

##### 3. 实现

```java
class Solution {
    
    private static int count = 0;

    public int movingCount(int m, int n, int k) {
        count = 0; // count是全局变量，每次调用movingCount方法时都要将其初始化
        int[][] visted = new int[m][n]; // 二维数组用来标识指定位置是否已被使用过
        put(visted, 0, 0, k);
        return count;
    }

    // 放置机器人
    private static void put(int[][]visted, int i, int j, int k){
        // 索引越界或当前位置已被访问过，无效，直接返回
        if(i<0 || i>=visted.length || j<0 || j>=visted[0].length || visted[i][j]==1){
            return;
        }
        // 如果索引i和j确定的位置可以放置机器人
        if(check(i, j, k) && visted[i][j]!=1){
            visted[i][j] = 1; // 表示此位置已被统计过
            count++;
            // 只需要向右或向下搜索，向上或向左一定是满足的
            //put(visted, i-1, j, k);
            put(visted, i+1,j,k);
            //put(visted, i, j-1, k);
            put(visted, i, j+1, k);
        }
    }

    // 判断索引i和j确定的位置，能否放置机器人
    private static boolean check(int i, int j, int k){
        int sum = sum(i) + sum(j);
        return sum <= k;
    }

    private static int sum(int i){
        int sum = 0;
        while(i > 0){
            sum += i % 10;
            i = i / 10;
        }
        return sum;
    }
}
```

算法分析：

- 时间复杂度：O(MN)。最差情况下，机器人需要访问所有位置。
- 空间复杂度：O(MN)。最坏情况下，机器人需要访问所有位置，辅助数组 `visted` 为 `m` 行、`n` 列。

---

