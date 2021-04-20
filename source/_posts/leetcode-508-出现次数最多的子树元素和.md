---
title: leetcode-508-出现次数最多的子树元素和
date: 2021-03-24 17:13:42
tags:
    - 树
    - 哈希表
categories: leetcode
---

```
//给你一个二叉树的根结点，请你找出出现次数最多的子树元素和。一个结点的「子树元素和」定义为以该结点为根的二叉树上所有结点的元素之和（包括结点本身）。 
//
// 你需要返回出现次数最多的子树元素和。如果有多个元素出现的次数相同，返回所有出现次数最多的子树元素和（不限顺序）。 
//
// 
//
// 示例 1： 
//输入: 
//
//   5
// /  \
//2   -3
// 
//
// 返回 [2, -3, 4]，所有的值均只出现一次，以任意顺序返回所有值。 
//
// 示例 2： 
//输入： 
//
//   5
// /  \
//2   -5
// 
//
// 返回 [2]，只有 2 出现两次，-5 只出现 1 次。 
//
// 
//
// 提示： 假设任意子树元素和均可以用 32 位有符号整数表示。 
// Related Topics 树 哈希表 
// 👍 108 👎 0


//leetcode submit region begin(Prohibit modification and deletion)

/**
 * Definition for a binary tree node.
 * public class TreeNode {
 * int val;
 * TreeNode left;
 * TreeNode right;
 * TreeNode(int x) { val = x; }
 * }
 */

//leetcode submit region end(Prohibit modification and deletion)
```

```java 
class Solution {
    int max = 1; // 记录最大出现次数
    Map<Integer, Integer> map = new HashMap<>(); // key-子树元素和，value-出现次数

    public int[] findFrequentTreeSum(TreeNode root) {
        if (root == null) return new int[0];
        sumTreeNode(root);
        List<Integer> list = new ArrayList<>();
        for (int i : map.keySet()) {
            // 取最大出现次数的值
            if (map.get(i) == max) {
                list.add(i);
            }
        }
        int index = 0;
        int[] result = new int[list.size()];
        while (index < result.length) {
            result[index] = list.get(index++);
        }
        return result;
    }

    public int sumTreeNode(TreeNode node) {
        if (node == null) return 0;
        // 递归求子树元素和
        int sum = node.val + sumTreeNode(node.left) + sumTreeNode(node.right);
        if (map.containsKey(sum)) {
            int temp = map.get(sum) + 1;
            map.put(sum, temp);
            max = Math.max(max, temp);
        } else {
            map.put(sum, 1);
        }
        return sum;
    }
}
```
    时间复杂度：O(N)
    空间复杂度：O(N)
    执行耗时:5 ms,击败了79.82% 的Java用户
	内存消耗:38.9 MB,击败了50.07% 的Java用户
	
	
二叉树题目
- 先序遍历（深度优先搜索）
- 中序遍历（深度优先搜索）（尤其二叉搜索树）
- 后序遍历（深度优先搜索）
- 层序遍历（广度优先搜索）