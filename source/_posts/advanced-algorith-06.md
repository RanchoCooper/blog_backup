---
title: 算法进阶(06) - 树、图
catalog: true
date: 2021-07-06 22:12:55
subtitle: Advanced Algorithm - Tree And Graph
author: Rancho
header-img:
tags:
    - 算法
---

# 树

## 二叉树的中序遍历
[LeetCode](https://leetcode-cn.com/problems/binary-tree-inorder-traversal/)

```go
/**
 * Definition for a binary tree node.
 * type TreeNode struct {
 *     Val int
 *     Left *TreeNode
 *     Right *TreeNode
 * }
 */
func inorderTraversal(root *TreeNode) []int {
    ans := make([]int, 0)
    inorder(root, &ans)
    return ans
}

func inorder(root *TreeNode, ans *[]int) {
    if root == nil {
        return
    }
    inorder(root.Left, ans)
    *ans = append(*ans, root.Val)
    inorder(root.Right, ans)
}
```

## N 叉树的前序遍历


### 解法一 递归
```go
/**
 * Definition for a Node.
 * type Node struct {
 *     Val int
 *     Children []*Node
 * }
 */

func preorder(root *Node) []int {
    ans := make([]int, 0)
    pre(root, &ans)
    return ans
}

func pre(root *Node, ans *[]int) {
    if root == nil {
        return
    }
    *ans = append(*ans, root.Val)
    for _, child := range root.Children {
        pre(child, ans)
    }
}
```

### 解法二 迭代
维护一个栈, 先将root入栈, 然后将其children逆序压入栈中, 然后不断出栈处理, 直到栈为空

```go
/**
 * Definition for a Node.
 * type Node struct {
 *     Val int
 *     Children []*Node
 * }
 */

func preorder(root *Node) []int {
    if root == nil {
        return []int{}
    }

    stack := make([]*Node, 0)
    ans := make([]int, 0)
    stack = append(stack, root)
    for ; len(stack) !=  0; {
        // pop
        current := stack[len(stack)-1]
        stack = stack[:len(stack)-1]

        ans = append(ans, current.Val)
        for i := len(current.Children); i > 0; i-- {
            stack = append(stack, current.Children[i-1])
        }
    }
    return ans
}
```

## N 叉树的层序遍历
[LeetCode](https://leetcode-cn.com/problems/n-ary-tree-level-order-traversal/)

### 解题思路
维护一个队列, 先将root入队, 不断取出队首元素(一层的全部Node列表)), 遍历其中的每个node, 将node的值和Children分别存储到切片中, 完成遍历后再将所有的node值切片插入到`ans`中
, 并将所有的`Children`入队, 直到队列为空
```go
/**
 * Definition for a Node.
 * type Node struct {
 *     Val int
 *     Children []*Node
 * }
 */

func levelOrder(root *Node) [][]int {
    if root == nil {
        return [][]int{}
    }
    ans := make([][]int, 0)
    queue := make([][]*Node, 0)

    queue = append(queue, []*Node{root})

    for len(queue) > 0 {
        // lpop
        nodes := queue[0]
        queue = queue[1:]

        levels := make([]int, 0)
        levelChildren := make([]*Node, 0)
        for _, node := range nodes {
            levels = append(levels, node.Val)
            levelChildren = append(levelChildren, node.Children...)
        }
        if len(levels) != 0 {
            queue = append(queue, levelChildren)
            ans = append(ans, levels)
        }
    }
    return ans
}
```

## 二叉树的序列化与反序列化

[LeetCode](https://leetcode-cn.com/problems/serialize-and-deserialize-binary-tree/)

序列化是将一个数据结构或者对象转换为连续的比特位的操作，进而可以将转换后的数据存储在一个文件或者内存中，同时也可以通过网络传输到另一个计算机环境，采取相反方式重构得到原数据。

请设计一个算法来实现二叉树的序列化与反序列化。这里不限定你的序列 / 反序列化算法执行逻辑，你只需要保证一个二叉树可以被序列化为一个字符串并且将这个字符串反序列化为原始的树结构。

提示: 输入输出格式与 LeetCode 目前使用的方式一致，详情请参阅 LeetCode 序列化二叉树的格式。你并非必须采取这种方式，你也可以采用其他的方法解决这个问题。

```go

```

## 从前序与中序遍历序列构造二叉树
[LeetCode](https://leetcode-cn.com/problems/construct-binary-tree-from-preorder-and-inorder-traversal/)

```go
/**
 * Definition for a binary tree node.
 * type TreeNode struct {
 *     Val int
 *     Left *TreeNode
 *     Right *TreeNode
 * }
 */
func buildTree(preorder []int, inorder []int) *TreeNode {
    return build(preorder, 0, len(preorder)-1, inorder, 0, len(inorder)-1)
}

func build(preorder []int, l1, r1 int, inorder []int, l2, r2 int) *TreeNode {
    if l1 > r1 {
        return nil
    }
    root := &TreeNode{
        Val: preorder[l1], 
    }
    mid := l2
    for inorder[mid] != root.Val {
        mid++
    }
    root.Left = build(preorder, l1+1, l1+mid-l2, inorder, l2, mid+1)
    root.Right = build(preorder, l1+mid-l2+1, r1, inorder, mid+1, r2)
    return root
}
```

## 二叉树的最近公共祖先

[LeetCode](https://leetcode-cn.com/problems/lowest-common-ancestor-of-a-binary-tree/)

给定一个二叉树, 找到该树中两个指定节点的最近公共祖先。

百度百科中最近公共祖先的定义为：“对于有根树 T 的两个节点 p、q，最近公共祖先表示为一个节点 x，满足 x 是 p、q 的祖先且 x 的深度尽可能大（一个节点也可以是它自己的祖先）。”

### 解题思路
首先维护一个map, 保存从每个节点到其父节点的映射(题目中每个节点的value不相同). 然后从q向上标记祖先节点, 放入redNotes中, 再对p求祖先, 当遇到第一个在redNotes中的祖先节点时, 即为最近的公共祖先

```go
/**
 * Definition for a binary tree node.
 * type TreeNode struct {
 *     Val int
 *     Left *TreeNode
 *     Right *TreeNode
 * }
 */

func lowestCommonAncestor(root, p, q *TreeNode) *TreeNode {
    if root == nil || root == p || root == q {
        return root
    }
    fathers := make(map[int]*TreeNode, 0)
    redNodes := make(map[int]bool, 0)

    var calcFather func(root *TreeNode)
    calcFather = func(root *TreeNode) {
        if root == nil {
            return
        }
        if root.Left != nil {
            fathers[root.Left.Val] = root
            calcFather(root.Left)
        }
        if root.Right != nil {
            fathers[root.Right.Val] = root
            calcFather(root.Right)
        }
    }
    calcFather(root)

    tmp := p

    for tmp != nil {
        redNodes[tmp.Val] = true
        tmp = fathers[tmp.Val]
    }

    tmp = q
    for tmp != nil {
        if redNodes[tmp.Val] {
            return tmp
        }
        tmp = fathers[tmp.Val]
    }
    return nil
}
```


# 图

## 课程表
[LeetCode](https://leetcode-cn.com/problems/course-schedule/)

你这个学期必须选修 numCourses 门课程，记为 0 到 numCourses - 1 。

在选修某些课程之前需要一些先修课程。 先修课程按数组 prerequisites 给出，其中 prerequisites[i] = [ai, bi] ，表示如果要学习课程 ai 则 必须 先学习课程  bi 。

    例如，先修课程对 [0, 1] 表示：想要学习课程 0 ，你需要先完成课程 1 。

请你判断是否可能完成所有课程的学习？如果可以，返回 true ；否则，返回 false 。

```go
func canFinish(numCourses int, prerequisites [][]int) bool {
    // 出边数组
    edges := make([][]int, numCourses)
    // 记录每个点的入度
    inDegree := make([]int, numCourses)
    // 可学习的课程数量
    learned := 0

    // 加边(x到y)
    var addEdge func(x, y int)
    addEdge = func(x, y int) {
        edges[x] = append(edges[x], y)
        inDegree[y]++
    }

    // 拓扑排序
    var topSort func()
    topSort = func() {
        queue := make([]int, 0)
        for i := 0; i < numCourses; i++ {
            // 将所有入度为0的点入队
            if inDegree[i] == 0 {
                queue = append(queue, i)
            }
        }

        for len(queue) != 0 {
            // pop
            x := queue[0]
            queue = queue[1:]
            learned++

            // x已学, 令其每个出边点的度数-1
            for _, y := range edges[x] {
                inDegree[y]--
                if inDegree[y] == 0 {
                    queue = append(queue, y)
                }
            }
        }
    }

    for _, pre := range prerequisites {
        addEdge(pre[1], pre[0])
    }
    topSort()

    return learned == numCourses
}
```


## 冗余连接
[LeetCode](https://leetcode-cn.com/problems/redundant-connection/description/)

在本问题中, 树指的是一个连通且无环的无向图。

输入一个图，该图由一个有着N个节点 (节点值不重复1, 2, ..., N) 的树及一条附加的边构成。附加的边的两个顶点包含在1到N中间，这条附加的边不属于树中已存在的边。

结果图是一个以边组成的二维数组。每一个边的元素是一对[u, v] ，满足 u < v，表示连接顶点u 和v的无向图的边。

返回一条可以删去的边，使得结果图是一个有着N个节点的树。如果有多个答案，则返回二维数组中最后出现的边。答案边 [u, v] 应满足相同的格式 u < v。

```go
func findRedundantConnection(input [][]int) []int {
    var n int
    // 出现最大的点数就是n
    for _, edge := range input {
        u, v := edge[0], edge[1]
        n = max(u, n)
        n = max(v, n)
    }
    var hasCycle bool
    // 出边数组
    edges := make([][]int, 0)
    visited := make([]bool, n+1)
    var dfs func(x, father int)
    dfs = func(x, father int) {
        visited[x] = true
        for _, y := range edges[x] {
            if y == father {
                continue
            }
            if visited[y] {
                hasCycle = true
            } else {
                dfs(y, x)
            }
        }
    }

    // 加边
    addEdge := func(x, y int) {
        edges[x] = append(edges[x], y)
    }

    // 初始化出边数组
    for i := 0; i <= n; i++ {
        edges = append(edges, make([]int, 0))
        visited[i] = false
    }

    // 加边
    for _, edge := range input {
        u, v := edge[0], edge[1]
        // 无向图看作双向边的有向图
        addEdge(u, v);
        addEdge(v, u);
        // 每加一条边，看图中是否多了环
        for i := 0; i <= n; i++ {
            visited[i] = false
        }
        dfs(u, -1)
        if hasCycle {
            return edge
        } 
    }
    return nil;
}

func max(a, b int) int {
    if a >= b {
        return a
    }
    return b
}
```















