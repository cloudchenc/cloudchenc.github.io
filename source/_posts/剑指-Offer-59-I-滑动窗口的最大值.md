---
title: 剑指 Offer 59 - I. 滑动窗口的最大值
date: 2020-07-22 15:39:43
tags:
    - 单调队列
    - 滑动窗口
categories: 剑指Offer
---

给定一个数组 nums 和滑动窗口的大小 k，请找出所有滑动窗口里的最大值。

时间复杂度
空间复杂度

执行用时：
14 ms
, 在所有 Java 提交中击败了
70.75%
的用户
内存消耗：
48.8 MB
, 在所有 Java 提交中击败了
100.00%
的用户

// 未形成窗口

// 形成窗口

```java
class Solution {
    public int[] maxSlidingWindow(int[] nums, int k) {
        if(nums.length == 0 || k == 0) return new int[0];
        Deque<Integer> deque = new LinkedList<>();
        int[] res = new int[nums.length - k + 1];
        for(int i = 0; i < k; i++){
            while(!deque.isEmpty() && deque.peekLast() < nums[i]) deque.removeLast();
            deque.addLast(nums[i]);
        }
        res[0] = deque.peekFirst();
        for(int i = k; i < nums.length; i++){
            if(deque.peekFirst() == nums[i - k]) deque.removeFirst();
            while(!deque.isEmpty() && deque.peekLast() < nums[i]) deque.removeLast();
            deque.addLast(nums[i]);
            res[i - k + 1] = deque.peekFirst();
        }
        return res;
    }
}
```

参考：
https://leetcode-cn.com/problems/hua-dong-chuang-kou-de-zui-da-zhi-lcof/solution/mian-shi-ti-59-i-hua-dong-chuang-kou-de-zui-da-1-6/