---
title: 剑指-Offer-60-n个骰子的点数
date: 2021-07-08 16:58:15
tags:
    - 动态规划
    - 数学
categories: 剑指Offer
---

```
//把n个骰子扔在地上，所有骰子朝上一面的点数之和为s。输入n，打印出s的所有可能的值出现的概率。 
//
// 
//
// 你需要用一个浮点数数组返回答案，其中第 i 个元素代表这 n 个骰子所能掷出的点数集合中第 i 小的那个的概率。 
//
// 
//
// 示例 1: 
//
// 输入: 1
//输出: [0.16667,0.16667,0.16667,0.16667,0.16667,0.16667]
// 
//
// 示例 2: 
//
// 输入: 2
//输出: [0.02778,0.05556,0.08333,0.11111,0.13889,0.16667,0.13889,0.11111,0.08333,0
//.05556,0.02778]
//
// 
//
// 限制： 
//
// 1 <= n <= 11 
// Related Topics 数学 动态规划 概率与统计 
// 👍 258 👎 0


//leetcode submit region begin(Prohibit modification and deletion)

//leetcode submit region end(Prohibit modification and deletion)
```

```java 
class Solution {
    public double[] dicesProbability(int n) {
        double[] dp = new double[6];
        Arrays.fill(dp, 1.0 / 6.0); // 初始化1个骰子的概率分布
        for (int i = 2; i <= n; i++) {
            double[] tmp = new double[5 * i + 1]; // n个骰子[n, n+1, ..., 6n]
            for (int j = 0; j < dp.length; j++) { // 在 'n-1' 个骰子的基础上，计算 'n' 个骰子的情况
                for (int k = 0; k < 6; k++) {
                    tmp[j + k] += dp[j] / 6.0;
                }
            }
            dp = tmp;
        }
        return dp;
    }
}
```
    时间复杂度：O(N ^ 2), 'N-1' 轮迭代中，每轮迭代的数量是 '6*6, 11*6, 16*6, 5(N-1)*6' 次
    空间复杂度：O(N)
    执行耗时:0 ms,击败了100.00% 的Java用户
    内存消耗:38.8 MB,击败了26.78% 的Java用户