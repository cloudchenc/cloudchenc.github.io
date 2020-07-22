---
title: 剑指 Offer 57. 和为s的两个数字
date: 2020-07-22 10:43:41
tags:
categories: 剑指Offer
---

输入一个递增排序的数组和一个数字s，在数组中查找两个数，使得它们的和正好是s。如果有多对数字的和等于s，则输出任意一对即可。

限制：  
- `1 <= nums.length <= 10^5`
- `1 <= nums[i] <= 10^6`

双指针法

时间复杂度 O(N)： N 为数组 nums 的长度；双指针共同线性遍历整个数组。
空间复杂度 O(1)： 变量 i, j 使用常数大小的额外空间。

执行用时：
2 ms
, 在所有 Java 提交中击败了
98.17%
的用户  
内存消耗：
56.8 MB
, 在所有 Java 提交中击败了
100.00%
的用户

```java
class Solution {
    public int[] twoSum(int[] nums, int target){
        int i = 0, j = nums.length - 1;
        while(i < j){
            int s = nums[i] + nums[j];
            if (s == target) return new int[]{nums[i], nums[j]};
            if(s > target) j--;
            if(s < target) i++;
        }
        return new int[0];
    }
}
```
