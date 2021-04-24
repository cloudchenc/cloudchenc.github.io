---
title: 剑指-Offer-35-复杂链表的复制
date: 2021-04-22 20:08:00
tags:
    - 链表
categories: 剑指Offer
---

```
//请实现 copyRandomList 函数，复制一个复杂链表。在复杂链表中，每个节点除了有一个 next 指针指向下一个节点，还有一个 random 指针指
//向链表中的任意节点或者 null。 
//
// 
//
// 示例 1： 
//
// 
//
// 输入：head = [[7,null],[13,0],[11,4],[10,2],[1,0]]
//输出：[[7,null],[13,0],[11,4],[10,2],[1,0]]
// 
//
// 示例 2： 
//
// 
//
// 输入：head = [[1,1],[2,1]]
//输出：[[1,1],[2,1]]
// 
//
// 示例 3： 
//
// 
//
// 输入：head = [[3,null],[3,0],[3,null]]
//输出：[[3,null],[3,0],[3,null]]
// 
//
// 示例 4： 
//
// 输入：head = []
//输出：[]
//解释：给定的链表为空（空指针），因此返回 null。
// 
//
// 
//
// 提示： 
//
// 
// -10000 <= Node.val <= 10000 
// Node.random 为空（null）或指向链表中的节点。 
// 节点数目不超过 1000 。 
// 
//
// 
//
// 注意：本题与主站 138 题相同：https://leetcode-cn.com/problems/copy-list-with-random-point
//er/ 
//
// 
// Related Topics 链表 
// 👍 195 👎 0


//leetcode submit region begin(Prohibit modification and deletion)
/*
// Definition for a Node.
class Node {
    int val;
    Node next;
    Node random;

    public Node(int val) {
        this.val = val;
        this.next = null;
        this.random = null;
    }
}
*/
//leetcode submit region end(Prohibit modification and deletion)
```

比较容易想到是哈希法，另外就是 拼接+拆分 法。

### 哈希表
```java 
class Solution {
    public Node copyRandomList(Node head) {
        if (head == null) return null;
        Map<Node, Node> map = new HashMap<>();
        Node curv = head;
        while (curv != null) {
            // 利用哈希表键值对的映射，保存新节点
            map.put(curv, new Node(curv.val));
            curv = curv.next;
        }

        curv = head;
        while (curv != null) {
            // 为新节点的next，random指针赋值
            map.get(curv).next = map.get(curv.next);
            map.get(curv).random = map.get(curv.random);
            curv = curv.next;
        }
        return map.get(head);
    }
}
```
    时间复杂度：O(N), N为链表长度，两次遍历链表，以空间换取了时间，在链表长度很长的情况下适用
    空间复杂度：O(N), N为链表长度
    执行耗时:0 ms,击败了100.00% 的Java用户
	内存消耗:37.9 MB,击败了80.49% 的Java用户

### 拼接+拆分
```java 
class Solution {
    public Node copyRandomList(Node head) {
        if (head == null) return null;

        Node curv = head;
        // 在 原链表 的每一个节点后 copy 一个新节点
        while (curv != null) {
            Node tempNode = new Node(curv.val);
            tempNode.next = curv.next;
            curv.next = tempNode; // 将当前节点的next指向新节点
            curv = tempNode.next; // 将当前节点指向原链表的下一个节点
        }

        curv = head;
        // 为每个新节点的random赋值
        while (curv != null) {
            if (curv.random != null) {
                // 注意这句 curv.random 是原链表节点的随机值，curv.random.next 才是对应的新节点
                curv.next.random = curv.random.next; 
            }
            curv = curv.next.next; // 将当前节点指向原链表的下一个节点
        }

        curv = head.next;
        Node prev = head;
        Node result = head.next;
        // 将当前链表，按照一个间隔一个顺序拆分，prev是head的指针，curv是result的指针
        // 利用 prev 还原 原链表 的每个节点
        // 并用 curv 把 新节点 串起来，组成copy后的 新链表
        while (curv.next != null) {
            prev.next = curv.next;
            prev = prev.next;
            curv.next = prev.next;
            curv = curv.next;
        }
        prev.next = null; // 将原链表的最后一个节点的next指针，重新指向null
        return result;
    }
}
```
    时间复杂度：O(N), N为链表长度，但存在三次遍历链表，如果链表长度过长，会导致耗时变多
    空间复杂度：O(1)，原地复制链表，不需要额外的辅助空间
    执行耗时:0 ms,击败了100.00% 的Java用户
    内存消耗:37.9 MB,击败了82.46% 的Java用户