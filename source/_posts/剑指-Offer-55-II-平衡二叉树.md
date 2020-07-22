---
title: 剑指 Offer 55 - II. 平衡二叉树
date: 2020-07-21 18:10:39
tags:
    - 树
    - 深度优先搜索
categories: 剑指Offer
---

### Description
输入一棵二叉树的根节点，判断该树是不是平衡二叉树。如果某二叉树中任意节点的左右子树的深度相差不超过1，那么它就是一棵平衡二叉树。

Tips:
1 <= 节点个数 <= 10000

### Thinking

### Solution

##### 后序遍历 + 剪枝 (从底到顶)

时间复杂度
空间复杂度

执行用时：
1 ms
, 在所有 Java 提交中击败了
99.91%
的用户  
内存消耗：
40.1 MB
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
    public boolean isBalanced(TreeNode root) {
        return recur(root) != -1;
    }

    int recur(TreeNode root){
        if(root == null) return 0;
        int left = recur(root.left);
        if(left == -1) return -1;
        int right = recur(root.right);
        if(right == -1) return -1;
        return Math.abs(left - right) < 2 ? Math.max(left, right) + 1 : -1;
    }
}
```

#### 先序遍历 + 判断深度 (从顶到底)

时间复杂度
空间复杂度

执行用时：
1 ms
, 在所有 Java 提交中击败了
99.91%
的用户
内存消耗：
39.9 MB
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
    public boolean isBalanced(TreeNode root) {
        if(root == null) return true;
        return Math.abs(depth(root.left) - depth(root.right)) <= 1
            && isBalanced(root.left)
            && isBalanced(root.right);
    }

    int depth(TreeNode root){
        if(root == null) return 0;
        return Math.max(depth(root.left), depth(root.right)) + 1;
    }  
}
```

参考：  
https://leetcode-cn.com/problems/ping-heng-er-cha-shu-lcof/solution/mian-shi-ti-55-ii-ping-heng-er-cha-shu-cong-di-zhi/