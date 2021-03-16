---
title: leetcode-234-回文链表
date: 2021-03-12 11:57:00
tags:
    - 链表
    - 双指针
categories: leetcode
---

```
//请判断一个链表是否为回文链表。 
//
// 示例 1: 
//
// 输入: 1->2
//输出: false 
//
// 示例 2: 
//
// 输入: 1->2->2->1
//输出: true
// 
//
// 进阶： 
//你能否用 O(n) 时间复杂度和 O(1) 空间复杂度解决此题？ 
// Related Topics 链表 双指针 
// 👍 890 👎 0

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

```java
class Solution {
    public boolean isPalindrome(ListNode head) {
        ListNode slow = head, fast = head; // 快慢指针
        ListNode prev = null; // 反转前半部分链表
        while (fast != null && fast.next != null) {
            ListNode temp = slow;
            slow = slow.next;
            fast = fast.next.next;
            temp.next = prev;
            prev = temp;
        }
        // 如果fast不为空，说明链表的长度是奇数个
        if (fast != null) {
            slow = slow.next; 
        }

        while (slow != null && prev != null) {
            if (slow.val != prev.val) {
                return false;
            }
            slow = slow.next;
            prev = prev.next;
        }
        return true;
    }
}
```
    时间复杂度：O(N)
    空间复杂度：O(1)
    执行耗时:1 ms,击败了100.00% 的Java用户
	内存消耗:40.8 MB,击败了95.27% 的Java用户
