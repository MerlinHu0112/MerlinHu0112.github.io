---
title: 剑指Offer题集（一）：链表问题集合
comments: true
top: false
date: 2020-03-24
tags:
	- Linked List
categories:
	- 剑指Offer
---

《剑指Offer》中涉及的链表问题集合。

总结：

- 对单链表的操作，善用两个指针！
- 对单链表的逆序操作，优先考虑递归或栈辅助。
- 对单链表的删除操作，常常要考虑设置dummyNode，以便操作头节点。
- 对单链表倒数第K个元素的操作，可用双指针，其中first指针要先行K步。
- 对循环单链表的操作，可用快慢指针。

---

<!--more-->

> 题目来源于《剑指Offer》第二版。同样的题目，在牛客网和LeetCode《剑指Offer》专区的表示和要求或许稍有差异，但算法实现是相同的。

---

#### 面试题06. 从尾到头打印链表【递归 or 栈】

题目：输入一个链表的头节点，从尾到头反过来返回每个节点的值（用数组返回）。

思路：

- 递归实现
- 栈辅助

实现：

*递归实现*

```java
class Solution {
    // 递归实现
    private ArrayList<Integer> list;
    
    public int[] reversePrint(ListNode head) {
        list = new ArrayList<>();
        recursion(head);

        int[] res = new int[list.size()];
        for(int i=0; i<list.size(); i++){
            res[i] = list.get(i);
        }
        return res;
    }

    private void recursion(ListNode node){
        if(node==null) return;
        recursion(node.next);
        list.add(node.val);
    }
}
```

*栈辅助*

```java
class Solution {
    // 借助栈先进后出的特点，实现单链表逆序打印
    public int[] reversePrint(ListNode head) {
        Stack<ListNode> stack = new Stack<>();
        ListNode current = head; // 当前节点指针
        int length = 0; // 统计节点个数
        while(current!=null){
            stack.push(current); // 压入节点指针
            current = current.next;
            length++;
        }

        int[] res = new int[length];
        for(int i=0; i<length; i++){
            res[i] = stack.pop().val;
        }
        return res;
    }

}
```

算法分析：

- 递归和栈辅助算法，时间复杂度均为：O(N)；
- 递归和栈辅助算法，空间复杂度均为：O(N)。

---

#### 面试题18(1). 删除链表的节点

题目：给定单向链表的头指针和一个节点指针，定义一个函数在O(1)时间内删除该节点。

思路：与[LeetCode上的改进版](https://leetcode-cn.com/problems/shan-chu-lian-biao-de-jie-dian-lcof/)和[LeetCode第237题-删除链表中的节点](https://leetcode-cn.com/problems/delete-node-in-a-linked-list/)不同，此题给定头指针和目标节点指针，且要求时间复杂度为O(1)。

此题分为三种情形：

- 链表仅有一个节点，且目标节点和头节点为该节点：直接置null。
- 目标节点是中间节点：类似LeetCode第237题的实现，用目标节点的下一节点元素值覆盖目标节点，并移除目标节点的下一节点即可。
- 目标节点是尾节点：遍历链表（**遍历操作时间复杂度不就是O(N)?**）

实现：

```java
public void deleteNode(ListNode head, ListNode deleteNode) {
	if(head==null || deleteNode==null){
        return;
    }
    // 链表仅有一个节点
    if(head==deleteNode){
        head = null; // 他人在此处写的是"head.next==null"。既然只有一个节点，其next比如为null，认为将head置null即可实现删除。
    }
    // 目标节点是中间节点
    else if(deleteNode.next!=null){
        ListNode nextNode = deleteNode.next;
        deleteNode.val = nextNode.val;
        deleteNode.next = nextNode.next;
    }
    // 目标节点是尾节点
    else{
        ListNode p = head;
        while(p.next!=deleteNode){
            p = p.next;
        }
        p.next = null;
    }
}
```

算法分析：

- 时间复杂度：平均为O(1)，坏情况下为O(N)；
- 空间复杂度：O(1)。

##### 另：LeetCode第237题-删除链表中的节点

题目限定：链表至少有两个节点，目标节点一定有效且不为尾节点。

```java
public void deleteNode(ListNode deleteNode) {
    ListNode pNext = deleteNode.next;
    deleteNode.val = pNext.val; 
    deleteNode.next = pNext.next;
}
```

---

#### 面试题18(2). 删除链表中的重复节点【双指针】

题目：在一个排序的链表中，存在重复的结点，请删除该链表中重复的结点，重复的结点不保留，返回链表头指针。

思路：注意需要删除全部重复节点。可采用双指针分别标识重复节点的起始节点的上一节点和重复节点的结束节点。注意，被标识的重复节点仅仅是当前最靠近链表头部的重复节点区域，其后可能还要其它值的重复节点，需要依次删除每一个重复的节点区域。

如 `1-2-2-2-3-3-4-5`，先标识出`2-2-2`并删除，再标识出`3-3`并删除，最终结果为`1-4-5`。

实现：

```java
public ListNode deleteDuplication(ListNode pHead){
    if(pHead==null){
        return pHead;
    }
    ListNode dummyNode = new ListNode(-1);
    dummyNode.next = pHead; // 便于删除头结点
    ListNode p = dummyNode; // p标识重复节点的起始节点的上一节点
    ListNode q; // q标识重复节点的结束节点
    while(p.next!=null){
        q = p.next;
        while(q.next!=null && q.next.val==q.val){
            q = q.next;
        }
        if(p.next!=q){
            // 有重复结点
            p.next = q.next;
        }else{
            p = q;
        }
    }
    return dummyNode.next;
}
```

算法分析：

- 时间复杂度：O(N)；
- 空间复杂度：O(1)。

---

#### 面试题22. 链表中倒数第k个节点【双指针】

题目：输入一个链表，输出该链表中倒数第k个节点。

思路：双指针，first指针比second指针先走k步；随后一起走，直至first指针为null，返回second指针指向的结点即可。

实现：

```java
public class Solution {
    public ListNode FindKthToTail(ListNode head,int k) {
        if(head==null || k<=0){
            return null;
        }
        ListNode first = head;
        ListNode second = head;
        // first指针先行k步
        int i;
        for(i=1; i<=k && first!=null; i++){
            first = first.next;
        }
        // k已超过链表长度，不合法
        if(first==null && (i<=k)){
            return null;
        }
        // first指针往后直至为null，second指针指向目标结点
        while(first!=null){
            first = first.next;
            second = second.next;
        }
        return second;
    }
}
```

算法分析：

- 时间复杂度为：O(N)；
- 空间复杂度为：O(1)。

---