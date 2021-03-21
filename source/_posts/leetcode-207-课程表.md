---
title: leetcode-207-课程表
date: 2021-03-16 11:54:02
tags:
    - 图
    - 拓扑排序
    - 广度优先搜索
    - 深度优先搜索
categories: leetcode
---

```
//你这个学期必须选修 numCourses 门课程，记为 0 到 numCourses - 1 。 
//
// 在选修某些课程之前需要一些先修课程。 先修课程按数组 prerequisites 给出，其中 prerequisites[i] = [ai, bi] ，表
//示如果要学习课程 ai 则 必须 先学习课程 bi 。 
//
// 
// 例如，先修课程对 [0, 1] 表示：想要学习课程 0 ，你需要先完成课程 1 。 
// 
//
// 请你判断是否可能完成所有课程的学习？如果可以，返回 true ；否则，返回 false 。 
//
// 
//
// 示例 1： 
//
// 
//输入：numCourses = 2, prerequisites = [[1,0]]
//输出：true
//解释：总共有 2 门课程。学习课程 1 之前，你需要完成课程 0 。这是可能的。 
//
// 示例 2： 
//
// 
//输入：numCourses = 2, prerequisites = [[1,0],[0,1]]
//输出：false
//解释：总共有 2 门课程。学习课程 1 之前，你需要先完成​课程 0 ；并且学习课程 0 之前，你还应先完成课程 1 。这是不可能的。 
//
// 
//
// 提示： 
//
// 
// 1 <= numCourses <= 105 
// 0 <= prerequisites.length <= 5000 
// prerequisites[i].length == 2 
// 0 <= ai, bi < numCourses 
// prerequisites[i] 中的所有课程对 互不相同 
// 
// Related Topics 深度优先搜索 广度优先搜索 图 拓扑排序 
// 👍 742 👎 0


//leetcode submit region begin(Prohibit modification and deletion)

//leetcode submit region end(Prohibit modification and deletion)
```

## 广度优先搜索

```
1. 统计课程安排图中每个节点的入度，生成 **入度表** `indegrees`。
2. 借助一个队列 `queue`，将所有入度为 *0* 的节点入队。
3. 当 `queue` 非空时，依次将队首节点出队，在课程安排图中删除此节点 `pre`：
   - 并不是真正从邻接表中删除此节点 `pre`，而是将此节点对应所有邻接节点 `cur` 的入度 *-1*，即 `indegrees[cur] -= 1`。
   - 当入度 *-1*后邻接节点 `cur` 的入度为 *0*，说明 `cur` 所有的前驱节点已经被 “删除”，此时将 `cur` 入队。
4. 在每次 `pre` 出队时，执行 `numCourses--`；
   - 若整个课程安排图是有向无环图（即可以安排），则所有节点一定都入队并出队过，即完成拓扑排序。换个角度说，若课程安排图中存在环，一定有节点的入度始终不为 *0*。
   - 因此，拓扑排序出队次数等于课程个数，返回 `numCourses == 0` 判断课程是否可以成功安排。
```

```java 
// BFS
class Solution {
    public boolean canFinish(int numCourses, int[][] prerequisites) {
        if (numCourses <= 0) return new int[0];

        List<List<Integer>> adjacency = new ArrayList<>(); // 构建邻接表
        for (int i = 0; i < numCourses; i++)
            adjacency.add(new ArrayList<>());

        int[] inDegrees = new int[numCourses]; // 保存节点的入度
        for (int[] p : prerequisites) {
            inDegrees[p[0]]++; // 记录每个节点的入度
            adjacency.get(p[1]).add(p[0]); // 保存出度的节点
        }

        Queue<Integer> queue = new LinkedList<>(); 
        for (int i = 0; i < numCourses; i++)
            if (inDegrees[i] == 0) queue.offer(i); // 保存入度为0的节点

        while (!queue.isEmpty()) {
            int pre = queue.poll();
            numCourses--; // 剪枝
            for (int cur : adjacency.get(pre))
                if (--inDegrees[cur] == 0) queue.offer(cur); // 剔除前驱节点，对应所有邻接节点入度-1
        }
        return numCourses == 0;
    }
}
```
    时间复杂度：O(N + M), 遍历一个图需要访问所有节点和所有临边，N 和 M 分别为节点数量和临边数量
    空间复杂度：O(N + M), 为建立邻接表所需额外空间，adjacency 长度为 N，并存储 M 条 临边的数据
    执行耗时:6 ms,击败了51.10% 的Java用户
    内存消耗:39.3 MB,击败了40.25% 的Java用户
    
## 深度优先搜索

原理是通过 DFS 判断图中是否有环。

```
1. 借助一个标志列表 `flags`，用于判断每个节点 `i` （课程）的状态：
   1. 未被 DFS 访问：`i == 0`；
   2. 已被**其他节点启动**的 DFS 访问：`i == -1`；
   3. 已被**当前节点启动**的 DFS 访问：`i == 1`。
2. 对 `numCourses` 个节点依次执行 DFS，判断每个节点起步 DFS 是否存在环，若存在环直接返回 *False*。DFS 流程；
   1. 终止条件：
      - 当 `flag[i] == -1`，说明当前访问节点已被其他节点启动的 DFS 访问，无需再重复搜索，直接返回 *True*。
      - 当 `flag[i] == 1`，说明在本轮 DFS 搜索中节点 `i` 被第 *2* 次访问，即 **课程安排图有环** ，直接返回 *False*。
   2. 将当前访问节点 `i` 对应 `flag[i]` 置 *1*，即标记其被本轮 DFS 访问过；
   3. 递归访问当前节点 `i` 的所有邻接节点 `j`，当发现环直接返回 *False*；
   4. 当前节点所有邻接节点已被遍历，并没有发现环，则将当前节点 `flag` 置为 *-1* 并返回 *True*。
3. 若整个图 DFS 结束并未发现环，返回 *True*。
```

```java 
// DFS
class Solution {
    public boolean canFinish(int numCourses, int[][] prerequisites) {
        List<List<Integer>> adjacency = new ArrayList<>();
        for (int i = 0; i < numCourses; i++)
            adjacency.add(new ArrayList<>());
        for (int[] p : prerequisites)
            adjacency.get(p[1]).add(p[0]);
        int[] flags = new int[numCourses];
        for (int i = 0; i < numCourses; i++)
            if (!dfs(adjacency, flags, i)) return false;
        return true;
    }

    public boolean dfs(List<List<Integer>> adjacency, int[] flags, int i) {
        if (flags[i] == 1) return false;
        if (flags[i] == -1) return true;
        flags[i] = 1;
        for (int j : adjacency.get(i))
            if (!dfs(adjacency, flags, j)) return false;
        flags[i] = -1;
        return true;
    }
}
```
    时间复杂度：O(N + M), 遍历一个图需要访问所有节点和所有临边，N 和 M 分别为节点数量和临边数量
    空间复杂度：O(N + M), 为建立邻接表所需额外空间，adjacency 长度为 N，并存储 M 条 临边的数据
    执行耗时:4 ms,击败了79.55% 的Java用户
    内存消耗:38.8 MB,击败了82.85% 的Java用户
    				