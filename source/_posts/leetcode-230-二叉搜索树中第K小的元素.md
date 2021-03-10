---
title: leetcode-230-二叉搜索树中第K小的元素
date: 2021-03-02 14:42:47
tags:
    - 二叉树
categories: leetcode
---

```
//给定一个二叉搜索树的根节点 root ，和一个整数 k ，请你设计一个算法查找其中第 k 个最小元素（从 1 开始计数）。 
//
// 
//
// 示例 1： 
//
// 
//输入：root = [3,1,4,null,2], k = 1
//输出：1
// 
//
// 示例 2： 
//
// 
//输入：root = [5,3,6,2,4,null,null,1], k = 3
//输出：3
// 
//
// 
//
// 
//
// 提示： 
//
// 
// 树中的节点数为 n 。 
// 1 <= k <= n <= 104 
// 0 <= Node.val <= 104 
// 
//
// 
//
// 进阶：如果二叉搜索树经常被修改（插入/删除操作）并且你需要频繁地查找第 k 小的值，你将如何优化算法？ 
// Related Topics 树 二分查找 
// 👍 355 👎 0


//leetcode submit region begin(Prohibit modification and deletion)

/**
 * Definition for a binary tree node.
 * public class TreeNode {
 * int val;
 * TreeNode left;
 * TreeNode right;
 * TreeNode() {}
 * TreeNode(int val) { this.val = val; }
 * TreeNode(int val, TreeNode left, TreeNode right) {
 * this.val = val;
 * this.left = left;
 * this.right = right;
 * }
 * }
 */

//leetcode submit region end(Prohibit modification and deletion)
```

二叉搜索树（BST）的中序遍历是升序排列

# Java

## 迭代 + 中序遍历
```java
// 迭代
class Solution {
    public int kthSmallest(TreeNode root, int k) {
        Deque<TreeNode> stack = new LinkedList<TreeNode>();
        while (root != null || !stack.isEmpty()) {
            while (root != null) {
                stack.push(root);
                root = root.left;
            }
            root = stack.pop();
            if (--k == 0) return root.val;
            root = root.right;
        }
        return 0;
    }
}
```
    
    时间复杂度：O(N), 为什么比递归慢了这么多？？？
    空间复杂度：O(N)
    执行耗时:1 ms,击败了45.85% 的Java用户  
    内存消耗:38 MB,击败了92.66% 的Java用户  
    
    Tips：
    时间复杂度是O(N), 常量级，其实本质是O(H + k), 其中H指的是树的高度. 
    由于我们开始遍历之前，要先向下达到叶，当树是一个平衡树时：复杂度为 O(logN + k); 当树是一个不平衡树时：复杂度为 O(N + k), 此时所有的节点都在左子树.
    空间复杂度 O(H + k)同理，平衡树时为 O(logN + k), 不平衡树时为 O(N + k)
    
## 递归 + 中序遍历
```java
// 递归
class Solution {
    int nums = 0;
    int res;
    public int kthSmallest(TreeNode root, int k) {
        inorderTravelsal(root, k);
        return res;
    }

    public void inorderTravelsal(TreeNode node, int k) {
        if (node == null) {
            return;
        }
        inorderTravelsal(node.left, k);
        nums++;
        if (nums == k) {
            res = node.val;
            return;
        }
        inorderTravelsal(node.right, k);
    }
}
```
    时间复杂度：O(N)
    空间复杂度：O(1)
    执行耗时:0 ms,击败了100.00% 的Java用户  
    内存消耗:38.1 MB,击败了84.59% 的Java用户
    
## 分治 + 中序遍历
```java
// 分治
class Solution {
    public int kthSmallest(TreeNode root, int k) {
        int n = nodeCount(root.left);
        if (n + 1 == k) {
            return root.val;
        } else if (n + 1 < k) {
            return kthSmallest(root.right, k - n - 1);
        } else {
            return kthSmallest(root.left, k);
        }
    }

    public int nodeCount(TreeNode node) {
        if (node == null) {
            return 0;
        }
        return 1 + nodeCount(node.left) + nodeCount(node.right);
    }
}
```
    时间复杂度：O(NlogN)
    空间复杂度：O(1)
    执行耗时:0 ms,击败了100.00% 的Java用户
    内存消耗:38.1 MB,击败了90.54% 的Java用户