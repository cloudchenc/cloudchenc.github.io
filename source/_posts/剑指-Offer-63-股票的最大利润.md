---
title: 剑指-Offer-63-股票的最大利润
date: 2021-07-09 17:48:55
tags:
    - 动态规划
categories: 剑指Offer
---

```
//假设把某股票的价格按照时间先后顺序存储在数组中，请问买卖该股票一次可能获得的最大利润是多少？ 
//
// 
//
// 示例 1: 
//
// 输入: [7,1,5,3,6,4]
//输出: 5
//解释: 在第 2 天（股票价格 = 1）的时候买入，在第 5 天（股票价格 = 6）的时候卖出，最大利润 = 6-1 = 5 。
//     注意利润不能是 7-1 = 6, 因为卖出价格需要大于买入价格。
// 
//
// 示例 2: 
//
// 输入: [7,6,4,3,1]
//输出: 0
//解释: 在这种情况下, 没有交易完成, 所以最大利润为 0。 
//
// 
//
// 限制： 
//
// 0 <= 数组长度 <= 10^5 
//
// 
//
// 注意：本题与主站 121 题相同：https://leetcode-cn.com/problems/best-time-to-buy-and-sell-s
//tock/ 
// Related Topics 数组 动态规划 
// 👍 133 👎 0


//leetcode submit region begin(Prohibit modification and deletion)

//leetcode submit region end(Prohibit modification and deletion)
```

## 动态规划

```java 
class Solution {
    public int maxProfit(int[] prices) {
        int[] dp = new int[prices.length];
        for (int i = 1; i < prices.length; i++) {
            int tmp = prices[i] - prices[i - 1] + dp[i - 1];
            if (tmp > 0) dp[i] = tmp;
        }

        int max = 0;
        for (int j : dp) {
            max = Math.max(max, j);
        }
        return max;
    }
}
```
    时间复杂度：O(N)
    空间复杂度：O(N)
    执行耗时:2 ms,击败了66.78% 的Java用户
    内存消耗:38 MB,击败了85.76% 的Java用户
    
## 遍历
    
```java 
class Solution {
    public int maxProfit(int[] prices) {
        int minPrice = Integer.MAX_VALUE;
        int maxProfit = 0;
        for (int i = 0; i < prices.length; i++) {
            int tmp = prices[i];
            if (tmp < minPrice) {
                minPrice = tmp;
            } else if (tmp - minPrice > maxProfit) {
                maxProfit = tmp - minPrice;
            }
        }
        return maxProfit;
    }
}
```
    时间复杂度：O(N)
    空间复杂度：O(1)
    执行耗时:1 ms,击败了98.33% 的Java用户
    内存消耗:38.1 MB,击败了69.96% 的Java用户