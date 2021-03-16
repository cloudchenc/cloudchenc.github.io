---
title: leetcode-24-两两交换链表中的节点
date: 2021-03-15 15:52:08
tags:
    - 链表
    - 递归
categories: leetcode
---

```
//给定一个链表，两两交换其中相邻的节点，并返回交换后的链表。 
//
// 你不能只是单纯的改变节点内部的值，而是需要实际的进行节点交换。 
//
// 
//
// 示例 1： 
//
// 
//输入：head = [1,2,3,4]
//输出：[2,1,4,3]
// 
//
// 示例 2： 
//
// 
//输入：head = []
//输出：[]
// 
//
// 示例 3： 
//
// 
//输入：head = [1]
//输出：[1]
// 
//
// 
//
// 提示： 
//
// 
// 链表中节点的数目在范围 [0, 100] 内 
// 0 <= Node.val <= 100 
// 
//
// 
//
// 进阶：你能在不修改链表节点值的情况下解决这个问题吗?（也就是说，仅修改节点本身。） 
// Related Topics 递归 链表 
// 👍 854 👎 0


//leetcode submit region begin(Prohibit modification and deletion)

/**
 * Definition for singly-linked list.
 * public class ListNode {
 * int val;
 * ListNode next;
 * ListNode() {}
 * ListNode(int val) { this.val = val; }
 * ListNode(int val, ListNode next) { this.val = val; this.next = next; }
 * }
 */

//leetcode submit region end(Prohibit modification and deletion)
```

```java
// 递归
class Solution {
    public ListNode swapPairs(ListNode head) {
        ListNode dummy = new ListNode(0);
        dummy.next = swap(head);
        return dummy.next;
    }

    public ListNode swap(ListNode node) {
        if (node == null || node.next == null) {
            return node;
        }

        ListNode curv = node.next; // 记录指针
        node.next = swap(curv.next); // 递归
        curv.next = node; // 反转

        return curv;
    }
}
```
    时间复杂度：O(N)
    空间复杂度：O(N)
    执行耗时:0 ms,击败了100.00% 的Java用户
	内存消耗:36 MB,击败了73.11% 的Java用户
	
```java
// 迭代
class Solution {
    public ListNode swapPairs(ListNode head) {
        ListNode dummy = new ListNode(0);
        dummy.next = head;

        ListNode tmp = dummy; // 记录指针
        while (tmp.next != null && tmp.next.next != null) {
            ListNode node1 = tmp.next; // 保存第一个节点
            ListNode node2 = node1.next; // 保存第二个节点
            tmp.next = node2; // 交换顺序
            node1.next = node2.next; // 交换顺序
            node2.next = node1; // 交换顺序
            tmp = node1; // 移动指针
        }
        return dummy.next;
    }
}
```
    时间复杂度：O(N)
    空间复杂度：O(1)
    执行耗时:0 ms,击败了100.00% 的Java用户
	内存消耗:36.1 MB,击败了45.86% 的Java用户