---
title: 剑指-Offer-48-最长不含重复字符的子字符串
date: 2021-07-07 10:39:15
tags:
    - 动态规划
    - 哈希表
categories: 剑指Offer
---

```
//请从字符串中找出一个最长的不包含重复字符的子字符串，计算该最长子字符串的长度。 
//
// 
//
// 示例 1: 
//
// 输入: "abcabcbb"
//输出: 3 
//解释: 因为无重复字符的最长子串是 "abc"，所以其长度为 3。
// 
//
// 示例 2: 
//
// 输入: "bbbbb"
//输出: 1
//解释: 因为无重复字符的最长子串是 "b"，所以其长度为 1。
// 
//
// 示例 3: 
//
// 输入: "pwwkew"
//输出: 3
//解释: 因为无重复字符的最长子串是 "wke"，所以其长度为 3。
//     请注意，你的答案必须是 子串 的长度，"pwke" 是一个子序列，不是子串。
// 
//
// 
//
// 提示： 
//
// 
// s.length <= 40000 
// 
//
// 注意：本题与主站 3 题相同：https://leetcode-cn.com/problems/longest-substring-without-rep
//eating-characters/ 
// Related Topics 哈希表 字符串 滑动窗口 
// 👍 232 👎 0


//leetcode submit region begin(Prohibit modification and deletion)

//leetcode submit region end(Prohibit modification and deletion)
```

## 动态规划 + 哈希表

```java 
// 空间换时间
class Solution {
    public int lengthOfLongestSubstring(String s) {
        Map<Character, Integer> dp = new HashMap<>();
        int res = 0, tmp = 0;
        for (int j = 0; j < s.length(); j++) {
            char c = s.charAt(j);
            int i = dp.getOrDefault(c, -1);
            dp.put(c, j);
            tmp = tmp < j - i ? tmp + 1 : j - i;
            res = Math.max(res, tmp);
        }
        return res;
    }
}
```
    时间复杂度：O(N)，其中 N 为字符串长度，动态规划需遍历计算 dp 列表
    空间复杂度：O(1)，字符的 ASCII 码范围为 0 ~ 127 ，哈希表 dic 最多使用 O(128) = O(1) 大小的额外空间
    执行耗时:7 ms,击败了82.29% 的Java用户
    内存消耗:38.5 MB,击败了43.02% 的Java用户
    
## 双指针 + 哈希表    

```java 
class Solution {
    public int lengthOfLongestSubstring(String s) {
        Map<Character, Integer> dic = new HashMap<>();
        int res = 0, i = -1;
        for (int j = 0; j < s.length(); j++) {
            char c = s.charAt(j);
            if (dic.containsKey(c)) {
                i = Math.max(i, dic.get(c));
            }
            dic.put(c, j);
            res = Math.max(res, j - i);
        }
        return res;
    }
}
```
    时间复杂度：O(N)，其中 N 为字符串长度，动态规划需遍历计算 dp 列表。
    空间复杂度：O(1)，字符的 ASCII 码范围为 0 ~ 127 ，哈希表 dic 最多使用 O(128) = O(1) 大小的额外空间
    执行耗时:6 ms,击败了90.90% 的Java用户
	内存消耗:38 MB,击败了98.34% 的Java用户
    
## 动态规划 + 线性遍历

```java 
// 时间换空间
class Solution {
    public int lengthOfLongestSubstring(String s) {
        int res = 0, tmp = 0;
        for (int j = 0; j < s.length(); j++) {
            int i = j - 1;
            while (i >= 0 && s.charAt(i) != s.charAt(j)) i--;
            tmp = tmp < j - i ? tmp + 1 : j - i;
            res = Math.max(res, tmp);
        }
        return res;
    }
}
```
    时间复杂度：O(N^2)，其中 N 为字符串长度，动态规划需遍历计算 dp 列表，占用 O(N) ；每轮计算 dp[j] 时搜索 i 需要遍历 j 个字符，占用 O(N) 。
    空间复杂度：O(1)，几个变量使用常数大小的额外空间
    执行耗时:13 ms,击败了14.91% 的Java用户
    内存消耗:38.2 MB,击败了93.24% 的Java用户
    
