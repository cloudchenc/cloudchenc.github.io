---
title: 剑指 Offer 54. 二叉搜索树的第k大节点
date: 2020-07-21 15:25:04
tags:
    - 树
categories: 剑指Offer
---

二叉搜索树的特点：

中序遍历为递增排列，所以中序遍历的倒序为递减排列

左-根-右变为右-根-左

时间复杂度：
空间复杂度：

执行用时：0 ms, 在所有 Java 提交中击败了100.00%的用户  
内存消耗：39.4 MB, 在所有 Java 提交中击败了100.00%的用户

```java
/**
 * Definition for a binary tree node.
 * public class TreeNode {
 *     int val;
 *     TreeNode left;
 *     TreeNode right;
 *     TreeNode(int x) { val = x; }
 * }
 */
class Solution {
    int res, k;
    public int kthLargest(TreeNode root, int k) {
        this.k = k;
        dfs(root);
        return res;
    }

    void dfs(TreeNode root){
        if(root == null || k == 0) return;
        dfs(root.right);
        if(--k == 0) res = root.val;
        dfs(root.left);
    }
}
```