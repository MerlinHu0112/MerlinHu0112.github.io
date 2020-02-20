---
title: test
date: 2020-01-16 16:03:47
tags:
---

```java
/**
 * 八皇后问题，基于递归的回溯思想，解决此问题
 * 2020-01-13
 * @author Merlin
 */
public class EightQueens {
	
	private static int[] checkerboard = new int[8]; // 棋盘的列投影，记录每一行的皇后存放的列索引值
	private static final int len = 8; // 棋盘的行或列长度为8
	private static final int queens = 8; // 皇后数，为8
	private int total = 0; // 记录总共的解法数，对八皇后问题，应为92
	
	public EightQueens() {}
	
	/**
	 * 递归地放置皇后到第i+1行合适的位置
	 * 放置第i+1行的皇后之后，需要判断其是否与之前的皇后冲突，若无，递归进入下一行；
	 * 若产生冲突，将其往后移动一列，避免冲突
```



我就试试不说话