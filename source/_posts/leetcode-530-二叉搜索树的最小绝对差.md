---
title: leetcode-530-二叉搜索树的最小绝对差
date: 2021-03-12 14:37:03
tags:
    - 树
categories: leetcode
---

```
//给你一棵所有节点为非负值的二叉搜索树，请你计算树中任意两节点的差的绝对值的最小值。 
//
// 
//
// 示例： 
//
// 输入：
//
//   1
//    \
//     3
//    /
//   2
//
//输出：
//1
//
//解释：
//最小绝对差为 1，其中 2 和 1 的差的绝对值为 1（或者 2 和 3）。
// 
//
// 
//
// 提示： 
//
// 
// 树中至少有 2 个节点。 
// 本题与 783 https://leetcode-cn.com/problems/minimum-distance-between-bst-nodes/ 
//相同 
// 
// Related Topics 树 
// 👍 235 👎 0

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
// 中序遍历（二叉搜索树中序遍历得到的值序列是递增有序的）
class Solution {

    int min = Integer.MAX_VALUE; // 差值的最小值
    TreeNode prev; // 保存前驱节点

    public int getMinimumDifference(TreeNode root) {
        inorder(root);
        return min;
    }

    public void inorder(TreeNode root) {
        //边界条件判断
        if (root == null)
            return;
        //左子树
        inorder(root.left);
        //对当前节点的操作
        if (prev != null)
            min = Math.min(min, root.val - prev.val);
        prev = root;
        //右子树
        inorder(root.right);
    }
}
```
    时间复杂度：O(N)，其中 N 为二叉搜索树节点的个数。每个节点在中序遍历中都会被访问一次且只会被访问一次，因此总时间复杂度为 O(N)。
    空间复杂度：O(N)，递归函数的空间复杂度取决于递归的栈深度，而栈深度在二叉搜索树为一条链的情况下会达到 O(N) 级别。
    执行耗时:0 ms,击败了100.00% 的Java用户
    内存消耗:38 MB,击败了79.05% 的Java用户