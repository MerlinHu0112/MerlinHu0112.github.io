---
title: 剑指Offer题集（二）：二叉树问题集合
comments: true
top: false
date: 2020-04-06 08:40:47
tags:
	- Binary Tree
categories:
	- 剑指Offer
---

《剑指Offer》中涉及的二叉树问题集合。

总结：

- 涉及二叉树的问题，离不开 DFS 和 BFS，即深度优先搜索和广度优先搜索。
- 常用的解题方法是递归和迭代，其中递归法更加直观，迭代法往往需要栈或队列的辅助。
- 要求处理一棵二叉树的遍历序列时，可先找出二叉树的根节点，再根据根节点将二叉树划分为左右子树并递归地处理左右子树。

<!--more-->

#### 面试题 07. 重建二叉树

##### 1. 题目

输入某二叉树的前序遍历和中序遍历的结果，请重建该二叉树。假设输入的前序遍历和中序遍历的结果中都不含重复的数字。

[力扣-《剑指Offer》-面试题07. 重建二叉树](https://leetcode-cn.com/problems/zhong-jian-er-cha-shu-lcof/)

[力扣-105. 从前序与中序遍历序列构造二叉树](https://leetcode-cn.com/problems/construct-binary-tree-from-preorder-and-inorder-traversal/)

##### 2. 思路

- 递归：易知，先序遍历得到的首个值是 root 节点的值，即可根据该元素值在中序遍历得到的数组中的位置，将其分为左右子树的节点，并分别递归地拼接左右子树。

##### 3. 实现

（1）递归实现：

> 需要注意的是，在递归建立左右子树的过程中，对左子树，注意 `preHi`；对右子树，注意 `preLo`。

```java
class Solution {
    private static int[] preAux = null; // 辅助数组，先序遍历
    private static int[] inAux = null; // 辅助数组，中序遍历

    public TreeNode buildTree(int[] preorder, int[] inorder) {
        if(preorder==null || inorder==null || preorder.length==0 || inorder.length==0){
            return null;
        }
        preAux = preorder;
        inAux = inorder;
        return helper(0, preAux.length-1, 0, inAux.length-1);
    }

    /**
     * @param preLo 先序遍历得到的数组的左边界
     * @param preHi 先序遍历得到的数组的右边界
     * @param inLo 中序遍历得到的数组的左边界
     * @param inHi 中遍历得到的数组的右边界
     * @return 建树完成，返回根节点内存地址
     */
    private static TreeNode helper(int preLo, int preHi, int inLo, int inHi){
        if(preLo > preHi || inLo > inHi){
            return null;
        }
        int rootVal = preAux[preLo]; // 当前子树的根结点
        TreeNode root = new TreeNode(rootVal);
        int index = inLo; // 根结点在中序遍历数组中的索引
        while(inAux[index]!=rootVal){
            // 结点元素值是唯一的，且一定存在
            index++;
        }
        // 在递归建立左右子树的过程中：对左子树，注意preHi的值；对右子树，注意preLo的值
        root.left = helper(preLo+1, preLo+(index-inLo), inLo, index-1);
        root.right = helper(preLo+(index-inLo)+1, preHi, index+1, inHi);
        return root;
    }
}
```

算法分析：

- 时间复杂度：O(N)，对于每个节点都有创建过程以及根据左右子树重建过程。
- 空间复杂度：O(N)，存储整棵树的开销。

（2）迭代实现

暂无，可参考[官方题解](https://leetcode-cn.com/problems/zhong-jian-er-cha-shu-lcof/solution/mian-shi-ti-07-zhong-jian-er-cha-shu-by-leetcode-s/)。

---

#### 面试题 26. 树的子结构

##### 1. 题目

[力扣-《剑指Offer》-面试题26. 树的子结构](https://leetcode-cn.com/problems/shu-de-zi-jie-gou-lcof/)

输入两棵二叉树 A 和 B，判断 B 是不是 A 的子结构。B 是 A 的子结构， 即 A 中有出现和 B 相同的结构和节点值。约定空树不是任意一个树的子结构。

##### 2. 实现

```java
/**
 * @Version 1.0
 * @Date 2020-04-15
 * 基于深度优先搜索
 */
public boolean isSubStructure(TreeNode A, TreeNode B) {
    if(A==null || B==null)
        return false; // 约定空树不是任意一个树的子结构
    return (dfs(A, B) || isSubStructure(A.left, B) || isSubStructure(A.right, B));
}

// 深度优先搜索，判断树B是否是树A的子结构
private static boolean dfs(TreeNode A, TreeNode B){
    if(A==null || A.val!=B.val)
        return false;
    if(B.left!=null && !dfs(A.left, B.left))
        return false;
    if(B.right!=null && !dfs(A.right, B.right))
        return false;
    return true;
}
```

算法分析：

- 时间复杂度：O(MN)，其中 M 和 N 分别是树 A 和 B 的节点数。先序遍历树 A，时间复杂度为 O(M)。在先序遍历树 A 的过程中调用 DFS 方法，时间复杂度为 O(N)。故算法的时间复杂度为 O(MN)。
- 空间复杂度：O(M)。当树 A 和 B 都退化为链表时，递归深度最大为 M，故空间复杂度为 O(M)。

---

#### 面试题 27. 二叉树的镜像

##### 1. 题目

请完成一个函数，输入一个二叉树，该函数输出它的镜像。即翻转二叉树。

[力扣-《剑指Offer》-面试题27. 二叉树的镜像](https://leetcode-cn.com/problems/er-cha-shu-de-jing-xiang-lcof/)

[力扣-226. 翻转二叉树](https://leetcode-cn.com/problems/invert-binary-tree/)

##### 2. 思路

- 递归：根据二叉树镜像的定义，通过**递归先序遍历**二叉树，互相交换左右子树，即可实现翻转二叉树。

##### 3. 实现

（1）递归实现

```java
class Solution {
    public TreeNode mirrorTree(TreeNode root) {
        if(root==null){
            return null;
        }
        helper(root);
        return root;
    }

    private static void helper(TreeNode root){
        if(root==null){
            return;
        }
        // 交换root节点的左右子树
        TreeNode temp = root.left;
        root.left = root.right;
        root.right = temp;
        // 递归修改左子树为镜像
        if(root.left!=null){
            helper(root.left);
        }
        // 递归修改右子树为镜像
        if(root.right!=null){
            helper(root.right);
        }
    }
}
```

算法分析：

- 时间复杂度：O(N)，需要遍历每一个节点。
- 空间复杂度：最坏的情况下，二叉树退化为链表，递归将被调用 N 次，故空间复杂度为 O(N)；在最好的情况下，二叉树是完全平衡的，高度为 O(logN)，故空间复杂度为 O(logN)。

（2）迭代实现

暂无，可参考[官方题解](https://leetcode-cn.com/problems/invert-binary-tree/solution/fan-zhuan-er-cha-shu-by-leetcode/)

---

#### 面试题 28. 对称的二叉树

##### 1. 题目

请实现一个函数，用来判断一棵二叉树是不是对称的。如果一棵二叉树和它的镜像一样，那么它是对称的。

##### 2. 思路

对于对称二叉树上的任意两个对称的非叶子节点 `left` 和 `right`，必然有如下关系成立：

- `left.val == right.val`；
- `left.left.val == right.right.val`；
- `left.right.val == right.left.val`。

对于任意两个对称的叶子节点，只有第一条关系式成立。

因此，递归地判断二叉树的左右子节点即可判断其是否为镜像对称的。

##### 3. 实现

（1）递归实现

```java
public class Solution {
    public boolean isSymmetric(TreeNode root) {
        if(root==null)
            return true;
        return recursive(root.left, root.right);
    }

    // 递归方法
    private boolean recursive(TreeNode left, TreeNode right){
        if(left==null && right==null)
            return true;
        if(left==null || right==null || left.val!=right.val)
            return false; // 只有一个子节点或左右子节点值不同时，一定不是镜像
        return (recursive(left.left, right.right) &&
                recursive(left.right, right.left));
    }
}
```

（2）迭代实现

迭代实现基于深度优先搜索，但插入队列顺序与 BFS 有差异

```java
import java.util.LinkedList;
import java.util.Queue;

class Solution {
    public boolean isSymmetric(TreeNode root) {
        Queue<TreeNode> queue = new LinkedList<>();
        // 开始时，根结点只有一个，将其存入两次
        queue.add(root);
        queue.add(root);
        // 当队列不为空时，每次取出两个连续的结点。
        // 当且仅当每次取出的两个结点值相等，才满足镜像对称
        while(!queue.isEmpty()) {
            TreeNode left = queue.poll();
            TreeNode right = queue.poll();
            // 结点A、B均为null，满足镜像对称，且需要跳过此次循环
            if(left==null && right==null) {
                continue;
            }
            // 只有一个子节点或左右子节点值不同时，一定不是镜像
            if(left==null || right==null || left.val!=right.val) {
                return false;
            }
            queue.add(left.left);
            queue.add(right.right);
            queue.add(left.right);
            queue.add(right.left);
        }
        return true;
    }
}
```

---

#### 面试题33. 二叉搜索树的后序遍历序列

##### 1. 题目

输入一个整数数组，判断该数组是不是某二叉搜索树的后序遍历结果。如果是则返回 `true`，否则返回 `false`。假设输入的数组的任意两个数字都互不相同。

[力扣-《剑指Offer》-面试题33.二叉搜索树的后序遍历序列](https://leetcode-cn.com/problems/er-cha-sou-suo-shu-de-hou-xu-bian-li-xu-lie-lcof/)

##### 2. 思路

- 二叉搜索树：左子树所有节点元素值小于根节点元素值，右子树所有节点元素值大于根节点元素值。
- 给定数组的最后一个元素就是根节点，因此可根据根节点将数组划分为左右子树序列，再递归地判断左右子树。

##### 3. 实现

```java
class Solution {
    public boolean verifyPostorder(int[] postorder) {
        if(postorder == null){
            return false;
        }
        /* 测试用例中，空数组返回true
        if(postorder == null || postorder.length <= 0){
            return false;
        }*/
        return helper(postorder, 0, postorder.length-1);
    }

    private boolean helper(int[] arr, int start, int end){
        // 递归终止条件
        if(start >= end){
            return true;
        }
        
        // 从索引start开始，寻找首个大于根节点元素值的元素
        int i = start;
        for(; i < end; i++){
            if(arr[i] > arr[end]){
                break;
            }
        }
        
        // 索引i将数组分为左右子树两部分
        int j = i;
        for(; j < end; j++){
            // 若右子树中存在小于根节点元素值的节点，直接返回false
            if(arr[j] < arr[end]){
                return false;
            }
        } 
        
        // 递归地判断左右子树
        return (helper(arr, start, i-1) && helper(arr, i, end-1));
    }
}
```

（有[资料](https://leetcode-cn.com/problems/er-cha-sou-suo-shu-de-hou-xu-bian-li-xu-lie-lcof/solution/mian-shi-ti-33-er-cha-sou-suo-shu-de-hou-xu-bian-6/)认为此算法的时间复杂度为 O(N^2)，空间复杂度为 O(N)，但我不是很理解。）

---

#### 面试题 55-I. 二叉树的深度

##### 1. 题目

输入一棵二叉树的根节点，求该树的深度。从根节点到叶节点依次经过的节点（含根、叶节点）形成树的一条路径，最长路径的长度为树的深度。

[力扣-《剑指Offer》-面试题55-I. 二叉树的深度](https://leetcode-cn.com/problems/er-cha-shu-de-shen-du-lcof/)

[力扣-104. 二叉树的最大深度](https://leetcode-cn.com/problems/maximum-depth-of-binary-tree/)

##### 2. 思路

- 递归：递归求解左右子树的最大深度，并选取较大者。
- 迭代：层序遍历，辅助队列每次存入当前层级所有节点，深度计数器加1。

##### 3. 实现

（1）递归实现

```java
class Solution {
    public int maxDepth(TreeNode root) {
        if(root==null){
            return 0;
        }
        return helper(root);
    }

    private static int helper(TreeNode root){
        if(root==null){
            return 0;
        }
        return Math.max(helper(root.left), helper(root.right))+1;
    }
}
```

算法分析：

- 时间复杂度：O(N)，需要遍历每一个节点。
- 空间复杂度：最坏的情况下，二叉树退化为链表，递归将被调用 N 次，故空间复杂度为 O(N)；在最好的情况下，二叉树是完全平衡的，高度为 O(logN)，故空间复杂度为 O(logN)。

（2）迭代实现

```java
class Solution {
    public int maxDepth(TreeNode root) {
        if(root==null){
            return 0;
        }      
        // BFS 广度优先搜索实现
        Queue<TreeNode> queue = new LinkedList<>();
        queue.add(root); // 存入根结点
        int level = 0; // 根结点所在层级为0
        while(!queue.isEmpty()){
            int levelSize = queue.size(); // 当前层结点数
            // 循环，取出当前层结点，并存入下一层结点
            for(int i=0; i<levelSize; i++){
                TreeNode node = queue.remove();
                if(node.left!=null){
                    queue.add(node.left);
                }
                if(node.right!=null){
                    queue.add(node.right);
                }
            }
            level++;
        }
        return level;
    }
}
```

算法分析：

- 时间复杂度：O(N)，需要遍历每一个节点。
- 空间复杂度：O(N)，队列存储遍历的每一个节点。

---

#### 面试题 68-I. 二叉树的最近公共祖先

##### 1. 题目

给定一个二叉树, 找到该树中两个指定节点的最近公共祖先。

> 最近公共祖先：对于有根树 T 的两个结点 p、q，最近公共祖先表示为一个结点 x，满足 x 是 p、q 的祖先且 x 的深度尽可能大（**一个节点也可以是它自己的祖先**）。

[力扣-《剑指Offer》-面试题68-I. 二叉树的最近公共祖先](https://leetcode-cn.com/problems/er-cha-shu-de-zui-jin-gong-gong-zu-xian-lcof/)

[力扣-236. 二叉树的最近公共祖先](https://leetcode-cn.com/problems/lowest-common-ancestor-of-a-binary-tree/)

##### 2. 思路

- 递归实现

##### 3. 实现

（1）递归实现

```java
class Solution {
    public TreeNode lowestCommonAncestor(TreeNode root, TreeNode p, TreeNode q) {
        if(root==null || root == p || root==q){
            return root;
        }

        TreeNode leftAncestor = lowestCommonAncestor(root.left, p, q); // 递归进入左子树查找
        TreeNode rightAncestor = lowestCommonAncestor(root.right, p, q); // 递归进入右子树查找

        if(leftAncestor==null){
            return rightAncestor; // 左子树没有，返回右子树
        }

        if(rightAncestor==null){
            return leftAncestor; // 右子树没有，返回左子树
        }
        return root; // 都没有，返回根结点
    }
}
```

算法分析：

- 时间复杂度：O(N)，最坏情况下需要遍历所有节点。
- 空间复杂度：O(N)，最坏情况下需要遍历所有节点。

---

#### 235. 二叉搜索树的最近公共祖先

##### 1. 题目

给定一个二叉搜索树, 找到该树中两个指定节点的最近公共祖先。

> 二叉搜索树的特性：左子树所有节点元素值不大于根节点元素值，右子树所有节点元素值不小于根节点元素值。

[力扣-235. 二叉搜索树的最近公共祖先](https://leetcode-cn.com/problems/lowest-common-ancestor-of-a-binary-search-tree/)

##### 2. 思路

- 递归：类似于上面的**二叉树的最近公共祖先**问题，且二叉搜索树的特性使问题得到简化。

##### 3. 实现

（1）递归实现

```java
class Solution {
    public TreeNode lowestCommonAncestor(TreeNode root, TreeNode p, TreeNode q) {
        if(root==null){
            return null;
        }
        // p和q节点在左子树
        if(p.val < root.val && q.val < root.val){
            return lowestCommonAncestor(root.left, p, q);
        }
        // p和q节点在右子树
        else if(p.val > root.val && q.val > root.val){
            return lowestCommonAncestor(root.right, p, q);
        }
        // 都没有，返回根节点
        return root;
    }
}
```

算法分析：

- 时间复杂度：O(N)。
- 空间复杂度：O(N)。

---

