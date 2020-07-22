---
title: 剑指 Offer 57 - II. 和为s的连续正数序列
date: 2020-07-22 13:46:01
tags:
categories: 剑指Offer
---

输入一个正整数 target ，输出所有和为 target 的连续正整数序列（至少含有两个数）。

序列内的数字由小到大排列，不同序列按照首个数字从小到大排列。

限制：  
- 1 <= target <= 10^5

滑动窗口法

执行用时：
4 ms
, 在所有 Java 提交中击败了
75.67%
的用户  
内存消耗：
37.6 MB
, 在所有 Java 提交中击败了
100.00%
的用户

时间复杂度 
空间复杂度 

```java
class Solution {
    public int[][] findContinuousSequence(int target) {
        List<int[]> res = new ArrayList<>();
        int i = 1, j = 1, sum = 0;
        while(i <= target / 2){
            if(sum < target){
                sum += j;
                j ++;
            } else if(sum > target) {
                sum -= i;
                i ++;
            } else {
                int[] tmp = new int[j - i];
                for(int k = i; k < j; k++){
                    tmp[k - i] = k; 
                }
                res.add(tmp);
                sum -= i;
                i ++;
            }
        }
        return res.toArray(new int[0][]);
    }
}
```

求根法

间隔法

双百法 ???
```java
class Solution {
    public int[][] findContinuousSequence(int target) {
        List<int[]> result = new ArrayList<>();
        int i = 1;

        while(target>0) {
            target -= i++;
            if(target>0 && target%i == 0) {
                int[] array = new int[i];
                for(int k = target/i, j = 0; k < target/i+i; k++,j++){
                    array[j] = k;
                }
                result.add(array);
            }
        }
        Collections.reverse(result);
        return result.toArray(new int[0][]);       
    }
}
```
