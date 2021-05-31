---
title: 剑指-Offer-44-数字序列中某一位的数字
date: 2021-05-28 11:03:25
tags:
    - 数学
categories: 剑指Offer
---

```
//数字以0123456789101112131415…的格式序列化到一个字符序列中。在这个序列中，第5位（从下标0开始计数）是5，第13位是1，第19位是4，
//等等。 
//
// 请写一个函数，求任意第n位对应的数字。 
//
// 
//
// 示例 1： 
//
// 输入：n = 3
//输出：3
// 
//
// 示例 2： 
//
// 输入：n = 11
//输出：0 
//
// 
//
// 限制： 
//
// 
// 0 <= n < 2^31 
// 
//
// 注意：本题与主站 400 题相同：https://leetcode-cn.com/problems/nth-digit/ 
// Related Topics 数学 
// 👍 128 👎 0

//leetcode submit region begin(Prohibit modification and deletion)

//leetcode submit region end(Prohibit modification and deletion)
```

```java 
class Solution {
    public int findNthDigit(int n) {
        int digit = 1;
        long start = 1;
        long count = 9;
        // 1.确定所求数位的所在数字的位数
        while (n > count) {
            n -= count;
            digit++;
            start *= 10;
            count = 9 * digit * start;
        }
        // 2.确定所求数位所在的数字
        long num = start + (n - 1) / digit;
        // 3.确定所求数位在 num 的哪一数位
        // ASCII - 0 = 真实值
        return Long.toString(num).charAt((n - 1) % digit) - '0';
    }
}
```
    执行耗时:0 ms,击败了100.00% 的Java用户
	内存消耗:35.3 MB,击败了32.53% 的Java用户
	
- **时间复杂度 O(logN)  ：** 所求数位 *n* 对应数字 *num* 的位数 *digit* 最大为 O(logN)  ；第一步最多循环 O(logN)  次；第三步中将 *num* 转化为字符串使用 O(logN)  时间；因此总体为 O(logN)  。 
- **空间复杂度 O(logN)  ：** 将数字 *num* 转化为字符串 `str(num)` ，占用 O(logN)  的额外空间。

