---
title: 剑指 Offer 58 - I. 翻转单词顺序
date: 2020-07-22 14:43:23
tags:
    - 字符串
categories: 剑指Offer
---

输入一个英文句子，翻转句子中单词的顺序，但单词内字符的顺序不变。为简单起见，标点符号和普通字母一样处理。例如输入字符串"I am a student. "，则输出"student. a am I"。

双指针法

时间复杂度 O(N)： 其中 N 为字符串 s 的长度，线性遍历字符串。  
空间复杂度 O(N)： 新建的 list(Python) 或 StringBuilder(Java) 中的字符串总长度 ≤N ，占用 O(N) 大小的额外空间。

执行用时：
3 ms
, 在所有 Java 提交中击败了
63.20%
的用户  
内存消耗：
39.7 MB
, 在所有 Java 提交中击败了
100.00%
的用户

```java
class Solution {
    public String reverseWords(String s) {
        s = s.trim();
        int j = s.length() - 1, i = j;
        StringBuilder res = new StringBuilder();
        while(i >= 0){
            while(i >= 0 && s.charAt(i) != ' ') i--;
            res.append(s.substring(i + 1, j + 1) + " ");
            while(i >= 0 && s.charAt(i) == ' ') i--;
            j = i;
        }
        return res.toString().trim();
    }
}
```

参考：  
https://leetcode-cn.com/problems/fan-zhuan-dan-ci-shun-xu-lcof/solution/mian-shi-ti-58-i-fan-zhuan-dan-ci-shun-xu-shuang-z/