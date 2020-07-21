---
title: 剑指 Offer 53 - II. 0～n-1中缺失的数字
date: 2020-07-21 14:50:14
tags:
    - 数组
    - 二分查找
categories: 剑指Offer
---

`一个长度为n-1的递增排序数组中的所有数字都是唯一的，并且每个数字都在范围0～n-1之内。在范围0～n-1内的n个数字中有且只有一个数字不在该数组中，请找出这个数字。`

时间复杂度：O(logN)： 二分法为对数级别复杂度。  
空间复杂度：O(1)： 几个变量使用常数大小的额外空间。  

执行用时：
0 ms
, 在所有 Java 提交中击败了
100.00%
的用户  
内存消耗：
40.7 MB
, 在所有 Java 提交中击败了
100.00%
的用户

```java
class Solution {
    public int missingNumber(int[] nums) {
        int i = 0, j = nums.length - 1;
        while(i <= j){
            int m = (i + j) / 2;
            if (nums[m] == m) i = m + 1;
            else j = m - 1;
        }
        return i;
    }
}
```

参考：  
