---
title: leetcode-21-合并两个有序链表
date: 2021-03-11 10:56:30
tags:
    - 链表
    - 递归
categories: leetcode
---

```
//将两个升序链表合并为一个新的 升序 链表并返回。新链表是通过拼接给定的两个链表的所有节点组成的。 
//
// 
//
// 示例 1： 
//
// 
//输入：l1 = [1,2,4], l2 = [1,3,4]
//输出：[1,1,2,3,4,4]
// 
//
// 示例 2： 
//
// 
//输入：l1 = [], l2 = []
//输出：[]
// 
//
// 示例 3： 
//
// 
//输入：l1 = [], l2 = [0]
//输出：[0]
// 
//
// 
//
// 提示： 
//
// 
// 两个链表的节点数目范围是 [0, 50] 
// -100 <= Node.val <= 100 
// l1 和 l2 均按 非递减顺序 排列 
// 
// Related Topics 递归 链表 
// 👍 1589 👎 0

//leetcode submit region begin(Prohibit modification and deletion)
/**
 * Definition for singly-linked list.
 * public class ListNode {
 *     int val;
 *     ListNode next;
 *     ListNode() {}
 *     ListNode(int val) { this.val = val; }
 *     ListNode(int val, ListNode next) { this.val = val; this.next = next; }
 * }
 */
//leetcode submit region end(Prohibit modification and deletion)
```

# Java

## 迭代
```java
class Solution {
    public ListNode mergeTwoLists(ListNode l1, ListNode l2) {
        ListNode dummy = new ListNode(0);
        ListNode curv = dummy; // 设置指针

        while (l1 != null && l2 != null) {
            if (l1.val >= l2.val) {
                curv.next = l2;
                l2 = l2.next;
            } else {
                curv.next = l1;
                l1 = l1.next;
            }
            curv = curv.next; // 移动指针
        }
        curv.next = l1 == null ? l2 : l1;
        return dummy.next;
    }
}
```
    时间复杂度：O(m + n)
    空间复杂度：O(1)
    执行耗时:0 ms,击败了100.00% 的Java用户
	内存消耗:37.9 MB,击败了49.70% 的Java用户
	
## 递归
```java
class Solution {
    public ListNode mergeTwoLists(ListNode l1, ListNode l2) {
        if (l1 == null) return l2;
        if (l2 == null) return l1;

        if (l1.val >= l2.val) {
            l2.next = mergeTwoLists(l1, l2.next);
            return l2;
        } else {
            l1.next = mergeTwoLists(l1.next, l2);
            return l1;
        }
    }
}
```
    时间复杂度：O(m + n)
    空间复杂度：O(m + n)
    执行耗时:0 ms,击败了100.00% 的Java用户
    内存消耗:37.8 MB,击败了66.97% 的Java用户
