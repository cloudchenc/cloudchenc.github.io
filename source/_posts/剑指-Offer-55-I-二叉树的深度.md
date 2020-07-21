---
title: 剑指 Offer 55 - I. 二叉树的深度
date: 2020-07-21 16:54:22
tags:
    - 树
    - 深度优先搜索
    - 广度优先搜索
categories: 剑指Offer
---

DFS 深度优先搜索

时间复杂度：
空间复杂度：

执行用时：
0 ms
, 在所有 Java 提交中击败了
100.00%
的用户  
内存消耗：
39.6 MB
, 在所有 Java 提交中击败了
100.00%
的用户

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
    public int maxDepth(TreeNode root) {
        if(root == null) return 0;
        return Math.max(maxDepth(root.left), maxDepth(root.right)) + 1;
    }
}
```

BFS 广度优先搜索

时间复杂度：
空间复杂度：

执行用时：
2 ms
, 在所有 Java 提交中击败了
11.51%
的用户  
内存消耗：
39.6 MB
, 在所有 Java 提交中击败了
100.00%
的用户

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
    public int maxDepth(TreeNode root) {
        if (root == null) return 0;

        List<TreeNode> queue = new LinkedList<>();
        queue.add(root);

        List<TreeNode> tmp;
        int res = 0;
        while (!queue.isEmpty()) {
            tmp = new LinkedList<>();
            for (TreeNode node : queue) {
                if (node.left != null) tmp.add(node.left);
                if (node.right != null) tmp.add(node.right);
            }
            queue = tmp;
            res++;
        }
        return res;
    }
}
```

参考：  
https://leetcode-cn.com/problems/er-cha-shu-de-shen-du-lcof/solution/mian-shi-ti-55-i-er-cha-shu-de-shen-du-xian-xu-bia/