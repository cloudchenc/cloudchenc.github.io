---
title: 剑指-Offer-38-字符串的排列
date: 2021-05-14 16:44:10
tags:
    - 回溯
categories: 剑指Offer
---

```
//输入一个字符串，打印出该字符串中字符的所有排列。 
//
// 
//
// 你可以以任意顺序返回这个字符串数组，但里面不能有重复元素。 
//
// 
//
// 示例: 
//
// 输入：s = "abc"
//输出：["abc","acb","bac","bca","cab","cba"]
// 
//
// 
//
// 限制： 
//
// 1 <= s 的长度 <= 8 
// Related Topics 回溯算法 
// 👍 266 👎 0


//leetcode submit region begin(Prohibit modification and deletion)

//leetcode submit region end(Prohibit modification and deletion)
```

```java 
class Solution {
    public String[] permutation(String s) {
        // 结果集合。这里之所有用set不用list
        // 主要原因是使用set可以主动去重，避免使用list每次判断是否存在某个元素
        // 其次题目不要求顺序
        Set<String> res = new HashSet<>();
        // 访问记录，便于过滤选择，做剪枝使用。
        boolean[] visited = new boolean[s.length()];
        // 开始遍历
        backtrack(s.toCharArray(), "", res, visited);
        // 将结果修改返回格式
        String[] realRes = new String[res.size()];
        int i = 0;
        for (String value : res) {
            realRes[i] = value;
            i++;
        }
        return realRes;
    }

    /**
      *
      * @param chars 题目给的参数
      * @param path 当前选择的字符串
      * @param res 已经选好的字符串集合
      * @param visited 访问记录
      * 
    **/
    private void backtrack(char[] chars, String path, Set<String> res, boolean[] visited) {
        // 结束条件，如果当前选择字符串和题目给定字符串的长度一致则结束
        if (path.length() == chars.length) {
            res.add(path);
            return;
        }
        // 遍历所有可以选择的条件
        for (int i = 0; i < chars.length; i++) {
            // 过滤掉已经选择的
            if (visited[i]) continue;
            // 当前选择
            visited[i] = true;
            // 递归进入下一层
            // 需要注意path这里也是做了选择和回退选择，因为是直接path + chars[i]，等递归结束之后path还是path，默认也就做了回退，比较取巧
            backtrack(chars, path + chars[i], res, visited);
            // 回退选择
            visited[i] = false;
        }
    }
}
```
    
- **时间复杂度 *O(N!N)* ：** *N* 为字符串 `s` 的长度；时间复杂度和字符串排列的方案数成线性关系，即复杂度为 *O(N!)* ；字符串拼接操作 `path + chars[i]` 使用 *O(N)* ；因此总体时间复杂度为 *O(N!N)* 。
- **空间复杂度 *O(N^2)* ：** 全排列的递归深度为 *N* ，系统累计使用栈空间大小为 *O(N)* ；递归中辅助 Set 累计存储的字符数量最多为 *N + (N-1) + ... + 2 + 1 = (N+1)N/2* ，即占用 *O(N^2)* 的额外空间。

    执行耗时:56 ms,击败了16.96% 的Java用户
    内存消耗:44.4 MB,击败了14.26% 的Java用户