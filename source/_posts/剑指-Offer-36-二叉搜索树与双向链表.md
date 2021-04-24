---
title: 剑指-Offer-36-二叉搜索树与双向链表
date: 2021-04-23 14:15:01
tags:
    - 树
    - 链表
categories: 剑指Offer
---

```
//输入一棵二叉搜索树，将该二叉搜索树转换成一个排序的循环双向链表。要求不能创建任何新的节点，只能调整树中节点指针的指向。 
//
// 
//
// 为了让您更好地理解问题，以下面的二叉搜索树为例： 
//
// 
//
// 
//
// 
//
// 我们希望将这个二叉搜索树转化为双向循环链表。链表中的每个节点都有一个前驱和后继指针。对于双向循环链表，第一个节点的前驱是最后一个节点，最后一个节点的后继是
//第一个节点。 
//
// 下图展示了上面的二叉搜索树转化成的链表。“head” 表示指向链表中有最小元素的节点。 
//
// 
//
// 
//
// 
//
// 特别地，我们希望可以就地完成转换操作。当转化完成以后，树中节点的左指针需要指向前驱，树中节点的右指针需要指向后继。还需要返回链表中的第一个节点的指针。 
//
// 
//
// 注意：本题与主站 426 题相同：https://leetcode-cn.com/problems/convert-binary-search-tree-
//to-sorted-doubly-linked-list/ 
//
// 注意：此题对比原题有改动。 
// Related Topics 分治算法 
// 👍 231 👎 0


//leetcode submit region begin(Prohibit modification and deletion)
/*
// Definition for a Node.
class Node {
    public int val;
    public Node left;
    public Node right;

    public Node() {}

    public Node(int _val) {
        val = _val;
    }

    public Node(int _val,Node _left,Node _right) {
        val = _val;
        left = _left;
        right = _right;
    }
};
*/
//leetcode submit region end(Prohibit modification and deletion)
```

======= 二叉搜索树的中序遍历是递增排列 =======

二叉搜索树转换成一个排序的循环双向链表

1. **排序链表：** 节点应从小到大排序，因此应使用 **中序遍历** “从小到大” 访问树的节点。
2. **双向链表：** 在构建相邻节点的引用关系时，设前驱节点 `pre` 和当前节点 `cur` ，不仅应构建 `pre.right = cur` ，也应构建 `cur.left = pre` 。
3. **循环链表：** 设链表头节点 `head` 和尾节点 `tail` ，则应构建 `head.left = tail` 和 `tail.right = head` 。

```java 
// 双指针遍历
class Solution {
    Node head = null; // 链表头节点
    Node tail = null; // 链表尾节点

    public Node treeToDoublyList(Node root) {
        if (root == null) return null;
        build(root);
        // 遍历完成后修改head前驱和tail后继，实现循环
        head.left = tail;
        tail.right = head;
        return head;
    }

    private void build(Node node) {
        if (node == null) return;
        
        // 中序遍历
        build(node.left);
        if (tail == null) {
            head = node; // 初始化头节点，只执行一次
        } else {
            tail.right = node; 
        }
        node.left = tail;
        tail = node; // 移动尾节点指针
        build(node.right);
    }
}
```
    时间复杂度：O(N), N为二叉树的节点数，中序遍历需要访问所有节点
    空间复杂度：O(N), 最差情况下，即树退化为链表时，递归深度达到 N，系统使用 O(N) 栈空间。
    执行耗时:0 ms,击败了100.00% 的Java用户
    内存消耗:37.9 MB,击败了53.66% 的Java用户
