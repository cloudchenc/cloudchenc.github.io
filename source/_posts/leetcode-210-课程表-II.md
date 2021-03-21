---
title: leetcode-210-课程表-II
date: 2021-03-16 15:19:26
tags:
    - 图
    - 拓扑排序
    - 广度优先搜索
    - 深度优先搜索
categories: leetcode
---

```
//现在你总共有 n 门课需要选，记为 0 到 n-1。 
//
// 在选修某些课程之前需要一些先修课程。 例如，想要学习课程 0 ，你需要先完成课程 1 ，我们用一个匹配来表示他们: [0,1] 
//
// 给定课程总量以及它们的先决条件，返回你为了学完所有课程所安排的学习顺序。 
//
// 可能会有多个正确的顺序，你只要返回一种就可以了。如果不可能完成所有课程，返回一个空数组。 
//
// 示例 1: 
//
// 输入: 2, [[1,0]] 
//输出: [0,1]
//解释: 总共有 2 门课程。要学习课程 1，你需要先完成课程 0。因此，正确的课程顺序为 [0,1] 。 
//
// 示例 2: 
//
// 输入: 4, [[1,0],[2,0],[3,1],[3,2]]
//输出: [0,1,2,3] or [0,2,1,3]
//解释: 总共有 4 门课程。要学习课程 3，你应该先完成课程 1 和课程 2。并且课程 1 和课程 2 都应该排在课程 0 之后。
//     因此，一个正确的课程顺序是 [0,1,2,3] 。另一个正确的排序是 [0,2,1,3] 。
// 
//
// 说明: 
//
// 
// 输入的先决条件是由边缘列表表示的图形，而不是邻接矩阵。详情请参见图的表示法。 
// 你可以假定输入的先决条件中没有重复的边。 
// 
//
// 提示: 
//
// 
// 这个问题相当于查找一个循环是否存在于有向图中。如果存在循环，则不存在拓扑排序，因此不可能选取所有课程进行学习。 
// 通过 DFS 进行拓扑排序 - 一个关于Coursera的精彩视频教程（21分钟），介绍拓扑排序的基本概念。 
// 
// 拓扑排序也可以通过 BFS 完成。 
// 
// 
// Related Topics 深度优先搜索 广度优先搜索 图 拓扑排序 
// 👍 366 👎 0


//leetcode submit region begin(Prohibit modification and deletion)

//leetcode submit region end(Prohibit modification and deletion)
```

```java 
class Solution {
    public int[] findOrder(int numCourses, int[][] prerequisites) {
        if (numCourses <= 0) return new int[0];

        List<List<Integer>> adj = new ArrayList<>();
        for (int i = 0; i < numCourses; i++) {
            adj.add(new ArrayList<>());
        }

        int[] inDegrees = new int[numCourses];
        for (int[] p : prerequisites) {
            inDegrees[p[0]]++;
            adj.get(p[1]).add(p[0]);
        }

        Queue<Integer> queue = new LinkedList<>();
        for (int i = 0; i < numCourses; i++)
            if (inDegrees[i] == 0) queue.offer(i);

        int[] res = new int[numCourses];
        int count = 0;
        while (!queue.isEmpty()) {
            int num = queue.poll();
            res[count++] = num;
            for (Integer j : adj.get(num)) {
                inDegrees[j]--;
                if (inDegrees[j] == 0) queue.offer(j);
            }
        }
        if (count == numCourses) {
            return res;
        }
        return new int[0];
    }
}
```
    时间复杂度：O(N + M), 遍历一个图需要访问所有节点和所有临边，N 和 M 分别为节点数量和临边数量
    空间复杂度：O(N + M), 为建立邻接表所需额外空间，adjacency 长度为 N，并存储 M 条 临边的数据
    执行耗时:5 ms,击败了78.05% 的Java用户
    内存消耗:39.7 MB,击败了46.32% 的Java用户
	
```java 
class Solution {
    int[] result;
    int index;
    public int[] findOrder(int numCourses, int[][] prerequisites) {
        List<List<Integer>> adjacency = new ArrayList<>();
        for (int i = 0; i < numCourses; i++)
            adjacency.add(new ArrayList<>());
        for (int[] p : prerequisites)
            adjacency.get(p[1]).add(p[0]);
        int[] flags = new int[numCourses];
        result = new int[numCourses];
        index = numCourses - 1;
        for (int i = 0; i < numCourses; i++)
            if (!dfs(adjacency, flags, i)) return new int[0];
        return result;
    }

    public boolean dfs(List<List<Integer>> adjacency, int[] flags, int i) {
        if (flags[i] == 1) return false;
        if (flags[i] == -1) return true;
        flags[i] = 1;
        for (int j : adjacency.get(i))
            if (!dfs(adjacency, flags, j)) return false;
        flags[i] = -1;
        result[index--] = i;
        return true;
    }
}
```
    时间复杂度：O(N + M), 遍历一个图需要访问所有节点和所有临边，N 和 M 分别为节点数量和临边数量
    空间复杂度：O(N + M), 为建立邻接表所需额外空间，adjacency 长度为 N，并存储 M 条 临边的数据
    执行耗时:3 ms,击败了99.43% 的Java用户
    内存消耗:40.1 MB,击败了12.83% 的Java用户