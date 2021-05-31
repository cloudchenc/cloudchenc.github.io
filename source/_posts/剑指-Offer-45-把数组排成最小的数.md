---
title: 剑指-Offer-45-把数组排成最小的数
date: 2021-05-28 16:24:16
tags:
    - 快排
categories: 剑指Offer
---

```
//输入一个非负整数数组，把数组里所有数字拼接起来排成一个数，打印能拼接出的所有数字中最小的一个。 
//
// 
//
// 示例 1: 
//
// 输入: [10,2]
//输出: "102" 
//
// 示例 2: 
//
// 输入: [3,30,34,5,9]
//输出: "3033459" 
//
// 
//
// 提示: 
//
// 
// 0 < nums.length <= 100 
// 
//
// 说明: 
//
// 
// 输出结果可能非常大，所以你需要返回一个字符串而不是整数 
// 拼接起来的数字可能会有前导 0，最后结果不需要去掉前导 0 
// 
// Related Topics 排序 
// 👍 225 👎 0


//leetcode submit region begin(Prohibit modification and deletion)

//leetcode submit region end(Prohibit modification and deletion)
```

```java 
class Solution {
    public String minNumber(int[] nums) {
        String[] strings = new String[nums.length];
        for (int i = 0; i < nums.length; i++) {
            strings[i] = String.valueOf(nums[i]);
        }
        quickSort(strings, 0, nums.length - 1);
        StringBuilder builder = new StringBuilder();
        for (String s : strings) {
            builder.append(s);
        }
        return builder.toString();
    }

    private void quickSort(String[] strings, int start, int end) {
        if (start >= end) return;
        int low = start, high = end;
        String anchor = strings[start];

        while (low < high) {
            while ((strings[high] + strings[start]).compareTo(strings[start] + strings[high]) >= 0 && low < high) high--;
            while ((strings[low] + strings[start]).compareTo(strings[start] + strings[low]) <= 0 && low < high) low++;
            anchor = strings[low];
            strings[low] = strings[high];
            strings[high] = anchor;
        }
        strings[low] = strings[start];
        strings[start] = anchor;
        // 此时 low == high
        quickSort(strings, start, low - 1);
        quickSort(strings, high + 1, end);
    }

}
```

    执行耗时:5 ms,击败了97.21% 的Java用户
    内存消耗:37.8 MB,击败了86.94% 的Java用户
    
- **时间复杂度 O(NlogN)  ：** *N* 为最终返回值的字符数量（ *strs* 列表的长度 <= N ）；使用快排或内置函数的平均时间复杂度为 O(NlogN)  ，最差为 *O(N^2)* 。
- **空间复杂度 *O(N)* ：** 字符串列表 *strs* 占用线性大小的额外空间。
