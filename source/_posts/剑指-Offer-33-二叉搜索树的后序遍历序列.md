---
title: 剑指-Offer-33-二叉搜索树的后序遍历序列
date: 2021-05-08 11:46:05
tags:
    - 树
categories: 剑指Offer
---

```
//输入一个整数数组，判断该数组是不是某二叉搜索树的后序遍历结果。如果是则返回 true，否则返回 false。假设输入的数组的任意两个数字都互不相同。 
//
// 
//
// 参考以下这颗二叉搜索树： 
//
//      5
//    / \
//   2   6
//  / \
// 1   3 
//
// 示例 1： 
//
// 输入: [1,6,3,2,5]
//输出: false 
//
// 示例 2： 
//
// 输入: [1,3,2,6,5]
//输出: true 
//
// 
//
// 提示： 
//
// 
// 数组长度 <= 1000 
// 
// 👍 256 👎 0


//leetcode submit region begin(Prohibit modification and deletion)

//leetcode submit region end(Prohibit modification and deletion)
```

### 递归

根据二叉搜索树的定义，左子树一定比根节点小，右子树一定比根节点大，以此递归，从数组即可判断是否为二叉搜索树，注意每次递归时需要把根节点去掉之后才能进入左右子树的递归。

（因为后序遍历，根节点在最后，而每棵子树的根节点是不一致的）

```java 
class Solution {
    public boolean verifyPostorder(int[] postorder) {
        return recur(postorder, 0, postorder.length - 1);
    }

    public boolean recur(int[] postorder, int i, int j) {
        if (i >= j) return true;
        int l = i; // 左子树下标
        while (postorder[l] < postorder[j]) l++; // 左子树小于根节点
        int r = l; // 右子树下标
        while (postorder[r] > postorder[j]) r++; // 右子树大于根节点

        return r == j && recur(postorder, i, l - 1) && recur(postorder, l, j - 1); // 尤其要注意右子树的根节点每次需要-1
    }
}
```
    时间复杂度：O(N^2), 每次调用 recur(i,j) 减去一个根节点，因此递归占用 O(N)；最差情况下（即当树退化为链表），每轮递归都需遍历树所有节点，占用 O(N)。
    空间复杂度：O(N), 最差情况下（即当树退化为链表）, 递归深度将达到 N。
    执行耗时:0 ms,击败了100.00% 的Java用户
	内存消耗:36 MB,击败了32.88% 的Java用户

### 辅助栈

- 遍历 “后序遍历的倒序” 会多次遇到递减节点 *r_i* ，若所有的递减节点 *r_i* 对应的父节点 *root* 都满足以上条件，则可判定为二叉搜索树。
- 根据以上特点，考虑借助 **单调栈** 实现：
  1. 借助一个单调栈 *stack* 存储值递增的节点；
  2. 每当遇到值递减的节点 *r_i* ，则通过出栈来更新节点 *r_i* 的父节点 *root* ；
  3. 每轮判断 *r_i* 和 *root*  的值关系：
     1. 若 *r_i > root* 则说明不满足二叉搜索树定义，直接返回 *false* 。
     2. 若 *r_i < root* 则说明满足二叉搜索树定义，则继续遍历。

```java 
class Solution {
    public boolean verifyPostorder(int[] postorder) {
        Stack<Integer> stack = new Stack<>();
        int root = Integer.MAX_VALUE;
        for (int i = postorder.length - 1; i >= 0; i--) {
            if (postorder[i] > root) return false; // 所有节点均需小于根节点
            while (!stack.isEmpty() && stack.peek() > postorder[i]) // 遇到递减节点，说明此时从某根节点的右子树遍历到了左子树
                root = stack.pop(); // 记录根节点
            stack.push(postorder[i]); // 存储递增节点
        }
        // 此时 stack 不一定为空
        return true;
    }
}
```
    时间复杂度：O(N), 遍历  postorder 所有节点，各节点均入栈 / 出栈一次，使用 O(N) 时间。
    空间复杂度：O(N), 最差情况下，单调栈 stack 存储所有节点，使用 O(N) 额外空间。
    执行耗时:1 ms,击败了21.30% 的Java用户
    内存消耗:35.9 MB,击败了45.40% 的Java用户