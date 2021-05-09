---
title: 剑指-Offer-34-二叉树中和为某一值的路径
date: 2021-05-08 17:13:52
tags:
    - 树
categories: 剑指Offer
---

```
//输入一棵二叉树和一个整数，打印出二叉树中节点值的和为输入整数的所有路径。从树的根节点开始往下一直到叶节点所经过的节点形成一条路径。 
//
// 
//
// 示例: 
//给定如下二叉树，以及目标和 target = 22， 
//
// 
//              5
//             / \
//            4   8
//           /   / \
//          11  13  4
//         /  \    / \
//        7    2  5   1
// 
//
// 返回: 
//
// 
//[
//   [5,4,11,2],
//   [5,8,4,5]
//]
// 
//
// 
//
// 提示： 
//
// 
// 节点总数 <= 10000 
// 
//
// 注意：本题与主站 113 题相同：https://leetcode-cn.com/problems/path-sum-ii/ 
// Related Topics 树 深度优先搜索 
// 👍 173 👎 0


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

先序遍历（根/左/右） + 路径记录（不满足条件时回溯到上一个节点）

这个题目的需要我们做的是判断那一路的节点之和等于target，在先根遍历的过程中，从上至下，然后又从下面的节点开始返回。
我们只需要从第一个节点开始累加，每遍历一个节点就就将这个节点加在list集合中，
当累加的结果等于target时，就可以将list集合加入到最终的结果中，并且需要将list集合中的最后一个元素删除，然后回溯到上一层，继续遍历其他路。
但是遍历到了最后一个节点后，也就是这个节点是叶子节点时，累加的值还没有等于target，那么就证明这条路行不通，需要开始遍历其他路，此时需要将list集合中的最后一个元素删除，因为需要回溯，回溯到上一层后list集合中最后一个元素就不需要了。

```java 
class Solution {
    LinkedList<List<Integer>> result = new LinkedList<>(); // 声明LinkedList，方便使用 removeLast 方法
    LinkedList<Integer> path = new LinkedList<>();

    public List<List<Integer>> pathSum(TreeNode root, int target) {
        recur(root, target);
        return result;
    }

    public void recur(TreeNode node, int target) {
        if (node == null) return;
        path.add(node.val);
        target -= node.val;
        if (target == 0 && node.left == null && node.right == null) // 当 node 为叶子节点时，且路径之和满足条件
            result.add(new LinkedList<>(path)); // 这里重新创建了集合，添加进入 result，避免 path 修改时，result 中的 path 也随之改变
        recur(node.left, target); // 递归左子树
        recur(node.right, target); // 递归右子树
        path.removeLast(); // 回溯至上一级
    }
}
```
    时间复杂度：O(N), N 为二叉树的节点数，先序遍历需要遍历所有节点
    空间复杂度：O(N), 最差情况下，即树退化为链表时，`path` 存储所有树节点，使用 O(N) 额外空间。
    执行耗时:1 ms,击败了100.00% 的Java用户
	内存消耗:38.9 MB,击败了40.71% 的Java用户