---
title: 剑指 Offer 52. 两个链表的第一个公共节点
tags:
    - 链表
    - 双指针
categories: 剑指Offer
---

双指针解法

假设链表A的长度为m+b, 链表B的长度为n+b, 公共部分为b。
走过m+n+b后，两个指针一定会相遇。

复杂度分析
- 时间复杂度：O(M+N)
- 空间复杂度：O(1)

执行用时：2 ms, 在所有 Java 提交中击败了27.37%的用户  
内存消耗：42.5 MB, 在所有 Java 提交中击败了100.00%的用户

```java
/**
 * Definition for singly-linked list.
 * public class ListNode {
 *     int val;
 *     ListNode next;
 *     ListNode(int x) {
 *         val = x;
 *         next = null;
 *     }
 * }
 */
public class Solution {
    public ListNode getIntersectionNode(ListNode headA, ListNode headB) {
        ListNode node1 = headA;
        ListNode node2 = headB;
        while(node1 != node2){
            node1 = node1 != null ? node1.next : headB;
            node2 = node2 != null ? node2.next : headA;
        }
        return node1;
    }
}
```

参考：  
https://leetcode-cn.com/problems/liang-ge-lian-biao-de-di-yi-ge-gong-gong-jie-dian-lcof/solution/shuang-zhi-zhen-fa-lang-man-xiang-yu-by-ml-zimingm/