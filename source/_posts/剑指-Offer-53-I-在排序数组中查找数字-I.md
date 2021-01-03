---
title: 剑指 Offer 53 - I. 在排序数组中查找数字 I
date: 2020-07-21 14:25:34
tags:
    - 数组
categories: 剑指Offer
---

数组有序，首先想到的就是二分查找

时间复杂度  
空间复杂度

执行用时：
0 ms
, 在所有 Java 提交中击败了
100.00%
的用户  
内存消耗：
42.7 MB
, 在所有 Java 提交中击败了
100.00%
的用户

```java
class Solution {
    public int search(int[] nums, int target) {
        return searchRight(nums, target) - searchRight(nums, target - 1);
    }

    int searchRight(int[] nums, int tar){
        int i = 0, j = nums.length - 1;
        while(i <= j){
            int m = (i + j) / 2;
            // 注意这里是 <= ，因为求是的右边界
            if(nums[m] <= tar) i = m + 1;
            else j = m - 1;
        }
        return i;
    }
}
```