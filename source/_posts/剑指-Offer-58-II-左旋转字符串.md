---
title: 剑指 Offer 58 - II. 左旋转字符串
date: 2020-07-22 15:18:56
tags:
    - 字符串
categories: 剑指Offer
---

字符串的左旋转操作是把字符串前面的若干个字符转移到字符串的尾部。  
请定义一个函数实现字符串左旋转操作的功能。  
比如，输入字符串"abcdefg"和数字2，该函数将返回左旋转两位得到的结果"cdefgab"。

限制：
+ 1 <= k < s.length <= 10000

时间复杂度 O(N)
空间复杂度 O(1)

执行用时：
0 ms
, 在所有 Java 提交中击败了
100.00%
的用户  
内存消耗：
40 MB
, 在所有 Java 提交中击败了
100.00%
的用户

```java
class Solution {
    public String reverseLeftWords(String s, int n) {
        return s.substring(n, s.length()) + s.substring(0, n);
    }
}
```

参考：  