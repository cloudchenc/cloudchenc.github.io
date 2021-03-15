---
title: 剑指-Offer-32-从上到下打印二叉树-I
date: 2021-03-11 14:44:28
tags:
categories:
---

```
//从上到下打印出二叉树的每个节点，同一层的节点按照从左到右的顺序打印。 
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
// 返回： 
//
// [3,9,20,15,7]
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
// 👍 64 👎 0


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
// 广度优先搜索（BFS）
class Solution {
    public int[] levelOrder(TreeNode root) {
        if (root == null) return new int[0];

        ArrayList<Integer> result = new ArrayList<>();
        Queue<TreeNode> queue = new LinkedList<>(); // 先进先出
        queue.offer(root);
        while (!queue.isEmpty()) {
            TreeNode node = queue.poll();
            result.add(node.val);
            if (node.left != null) queue.offer(node.left); // 换行
            if (node.right != null) queue.offer(node.right); // 换行
        }
        int[] res = new int[result.size()];
        for (int i = 0; i < result.size(); i++) {
            res[i] = result.get(i);
        }
        return res;
    }
}
```
    时间复杂度：O(N)
    空间复杂度：O(N)
    执行耗时:1 ms,击败了99.78% 的Java用户
	内存消耗:38.7 MB,击败了32.52% 的Java用户