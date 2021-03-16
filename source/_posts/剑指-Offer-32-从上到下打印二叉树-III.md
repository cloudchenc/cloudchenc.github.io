---
title: 剑指-Offer-32-从上到下打印二叉树-III
date: 2021-03-11 15:08:21
tags:
    - 树
    - 广度优先搜索
categories: 剑指Offer
---

```
//请实现一个函数按照之字形顺序打印二叉树，即第一行按照从左到右的顺序打印，第二层按照从右到左的顺序打印，第三行再按照从左到右的顺序打印，其他行以此类推。 
//
// 
//
// 例如: 
//给定二叉树: [3,9,20,null,null,15,7], 
//
//     3
//   / \
//  9  20
//    /  \
//   15   7
// 
//
// 返回其层次遍历结果： 
//
// [
//  [3],
//  [20,9],
//  [15,7]
//]
// 
//
// 
//
// 提示： 
//
// 
// 节点总数 <= 1000 
// 
// Related Topics 树 广度优先搜索 
// 👍 78 👎 0

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

# Java

## 层序遍历 + 双端队列
```java
class Solution {
    public List<List<Integer>> levelOrder(TreeNode root) {
        if (root == null) return new ArrayList<>();

        List<List<Integer>> res = new ArrayList<>();
        Queue<TreeNode> queue = new LinkedList<>();
        queue.offer(root);
        while (!queue.isEmpty()) {
            LinkedList<Integer> tmp = new LinkedList<>();
            // 由于queue出队，导致size减小，所以这里使用倒序for循环
            for (int i = queue.size(); i > 0; i--) {
                TreeNode node = queue.poll();
                if (res.size() % 2 == 1) {
                    tmp.addFirst(node.val); // 本层是偶数层
                } else {
                    tmp.addLast(node.val); // 本层是奇数层
                }
                if (node.left != null) queue.offer(node.left);
                if (node.right != null) queue.offer(node.right);
            }
            res.add(tmp);
        }
        return res;
    }
}
```
    时间复杂度：O(N)
    空间复杂度：O(N)
    执行耗时:1 ms,击败了99.74% 的Java用户
	内存消耗:38.8 MB,击败了19.96% 的Java用户
	
## 层序遍历 + 双端队列（奇偶层逻辑分离）
```java
class Solution {
    public List<List<Integer>> levelOrder(TreeNode root) {
        if (root == null) return new ArrayList<>();

        List<List<Integer>> res = new ArrayList<>();
        Deque<TreeNode> deque = new LinkedList<>(); // 双端队列
        deque.add(root);
        while (!deque.isEmpty()) {
            List<Integer> tmp = new LinkedList<>();
            for (int i = deque.size(); i > 0; i--) {
                // 这里奇数层从左向右打印，要与偶数层相反
                TreeNode node = deque.removeFirst();
                tmp.add(node.val);
                // 本层顺序打印，所以顺序添加下一层到尾部
                if (node.left != null) deque.addLast(node.left);
                if (node.right != null) deque.addLast(node.right);
            }
            res.add(tmp);
            if (deque.isEmpty()) break; // 后续没有节点则提前退出

            tmp = new LinkedList<>();
            for (int i = deque.size(); i > 0; i--) {
                // 这里偶数层从右想左打印，要与奇数层相反
                TreeNode node = deque.removeLast();
                tmp.add(node.val);
                // 本层逆序打印，所以逆序添加下一层到头部
                if (node.right != null) deque.addFirst(node.right);
                if (node.left != null) deque.addFirst(node.left);
            }
            res.add(tmp);
        }
        return res;
    }
}
```
    时间复杂度：O(N)
    空间复杂度：O(N)
    执行耗时:1 ms,击败了99.74% 的Java用户
    内存消耗:38.8 MB,击败了21.49% 的Java用户