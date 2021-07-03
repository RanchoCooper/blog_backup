---
title: 算法进阶(05) - 递归、分治
catalog: true
date: 2021-07-03 07:44:55
subtitle: Advanced Algorith - Recursion And Divide
author: Rancho
header-img:
tags:
    - 算法
---
# 基础知识

## 递归的时间复杂度    

* 指数型: k^n (子集, 大体积背包问题)
* 排列型: n! (全排列, 旅行商, N皇后)
* 组合型: n!/(m!(n-m)!) (组合选数)

# 递归

## 子集
[LeetCode](https://leetcode-cn.com/problems/subsets/)

给你一个整数数组 nums ，数组中的元素 互不相同 。返回该数组所有可能的子集（幂集）。解集 不能 包含重复的子集。你可以按 任意顺序 返回解集。
### 解题思路
按数组index递归找出子集, 把找出的子集放入全局变量中. 每个元素在或不在子集中, 所以要分两种情况进行递归: 元素不在子集中, 直接index++; 元素在子集中, 将该元素添加到全局变量set中, 并且在递归后还原状态. 终止条件为index = nums.size
```go
func subsets(nums []int) [][]int {    
    ans := make([][]int, 0)    
    set := make([]int, 0)
    var findSubSet func([]int, int) 
    findSubSet = func(nums []int, index int) { 
        // 递归终止条件       
        if index == len(nums) {
            // 完成一次递归, 将当前set加入ans中            
            //注意这里需要对set进行一次copy 
            ans = append(ans, append([]int(nil), set...))
            return 
        }  
        // 元素不在子集中, 继续递归      
        findSubSet(nums, index+1) 
        // 元素在子集中, 将当前元素加入set中      
        set = append(set, nums[index]) 
        findSubSet(nums, index+1)
        // 还原set      
        set = set[:len(set)-1] 
    } 
    findSubSet(nums, 0) 
    return ans}
```

## 组合
[LeetCode](https://leetcode-cn.com/problems/combinations/) 
给定两个整数 n 和 k，返回 1 ... n 中所有可能的 k 个数的组合。示例:输入: n = 4, k = 2输出:[  [2,4],  [3,4],  [2,3],  [1,2],  [1,3],  [1,4],]

### 解题思路
这题与上道题类似, 组合是子集的一部分, 即集合内的元素数量刚好为k

```go
func combine(n int, k int) [][]int { 
    ans := make([][]int, 0)  
    set := make([]int, 0)  
    var findCombine func(int)  
    findCombine = func(index int) { 
        if len(set) > k || index + n - index + 1 < k {   
            // 当前元素个数已大于k, 或者即使剩余元素全部选上, 都不够k个            
            //提前退出      
            return
        }  
        if index == n + 1 {    
            // 子集中元素个数为k         
            if len(set) == k {     
                ans = append(ans, append([]int{}, set...))  
            }   
            return     
        }
        findCombine(index + 1)  
        set = append(set, index)
        findCombine(index + 1)    
        set = set[:len(set) - 1] 
    }
    findCombine(1)
    return ans
}
```

## 全排列
给定一个不含重复数字的数组 nums ，返回其 所有可能的全排列 。你可以 按任意顺序 返回答案。示例 1：输入：nums = [1,2,3]输出：[[1,2,3],[1,3,2],[2,1,3],[2,3,1],[3,1,2],[3,2,1]]示例 2：输入：nums = [0,1]输出：[[0,1],[1,0]]示例 3：输入：nums = [1]输出：[[1]]

### 解题思路
同上, 递归遍历每一层, 不过每一层不在是选择或不选择, 而是nums中的n个数作为候选项, 因此在递归内部需要迭代nums中的每个元素

```go
func permute(nums []int) [][]int {
    ans := make([][]int, 0)    
    used := make(map[int]bool, 0) 
    cur := make([]int, 0)  
    var findPermute func(int)  
    findPermute = func(index int) {  
        if index == len(nums) {      
            ans = append(ans, append([]int{}, cur...))      
            return      
        } 
        for _, i := range nums {  
            if used[i] == false {    
                used[i] = true      
                cur = append(cur, i)     
                findPermute(index+1)  
                used[i] = false  
                cur = cur[:len(cur) - 1]      
            }        
        } 
    } 
    findPermute(0)   
    return ans
}
```

## 翻转二叉树
[LeetCode](https://leetcode-cn.com/problems/invert-binary-tree/) 

翻转一棵二叉树。

```go
/**
 * Definition for a binary tree node. 
 * type TreeNode struct { 
 *     Val int 
 *     Left *TreeNode 
 *     Right *TreeNode
 * } 
 */
func invertTree(root *TreeNode) *TreeNode { 
    if root == nil {     
        return nil   
    } 
    invertTree(root.Left)  
    invertTree(root.Right)   
    root.Left, root.Right = root.Right, root.Left 
    return
    root}
```

## 验证二叉搜索树
[LeetCode](https://leetcode-cn.com/problems/validate-binary-search-tree/)
给定一个二叉树，判断其是否是一个有效的二叉搜索树。假设一个二叉搜索树具有如下特征：


* 节点的左子树只包含小于当前节点的数。
* 节点的右子树只包含大于当前节点的数。
* 所有左子树和右子树自身必须也是二叉搜索树。

```go
/** 
 * Definition for a binary tree node. 
 * type TreeNode struct { 
 *     Val int 
 *     Left *TreeNode 
 *     Right *TreeNode 
 * } 
 */
import "math"

type NodeInfo struct {
    min, max int64    
    isValid bool
}

func isValidBST(root *TreeNode) bool {
    r := isValid(root)  
    return r.isValid
}

func isValid(root *TreeNode) *NodeInfo { 
    if root == nil {  
        return &NodeInfo{       
            min: math.MaxInt64,
            max: math.MinInt64,
            isValid: true,  
        } 
    } 
    left := isValid(root.Left)  
    right := isValid(root.Right)
    var result NodeInfo 
    
    result.max = int64(math.Max(float64(root.Val), math.Max(float64(left.max), float64(right.max))))  
    result.min = int64(math.Min(float64(root.Val), math.Min(float64(left.min), float64(right.min))))  
    result.isValid = left.isValid && right.isValid &&              
                        left.max < int64(root.Val) && int64(root.Val) < right.min 
    return &result}
```

## 二叉树的最大深度
[LeetCode](https://leetcode-cn.com/problems/maximum-depth-of-binary-tree/)
给定一个二叉树，找出其最大深度。二叉树的深度为根节点到最远叶子节点的最长路径上的节点数。说明: 叶子节点是指没有子节点的节点。

### 解法一
自底向上统计信息 (分治思想)最大深度 = max(左子树最大深度+右子树最大深度) + 1

```go
/** 
 * Definition for a binary tree node. 
 * type TreeNode struct {
 *     Val int 
 *     Left *TreeNode 
 *     Right *TreeNode 
 * } 
 */ 
import "math"

func maxDepth(root *TreeNode) int {    
    if root == nil {  
        return 0   
    }
    return int(math.Max(float64(maxDepth(root.Left)), float64(maxDepth(root.Right)))) + 1}
```

### 解法二
自顶向下维护信息, 一种写法是维护全局变量, 每递归一层, 令其+1, 需要注意还原现场; 另一种写法是通过函数参数传入, 这种写法每次调用递归函数时, 都会把该参数压栈, 空间上回多消耗一些, 但是更容易编写

```go
/**
 * Definition for a binary tree node. 
 * type TreeNode struct { 
 *     Val int 
 *     Left *TreeNode
 *     Right *TreeNode 
 * }
 */
import "math"

// 写法一
func maxDepth(root *TreeNode) int {
    var ans int    
    var depth func(*TreeNode, int)
    depth = func (root *TreeNode, current int)  {
        if root == nil {           
            return
        } 
        ans = int(math.Max(float64(ans), float64(current)))
        depth(root.Left, current+1)
        depth(root.Right, current+1) 
    }
    depth(root, 1)
    return ans
}

// 写法二
func maxDepth(root *TreeNode) int { 
    var current, ans int 
    var depth func(*TreeNode)
    depth = func (root *TreeNode)  {        
        if root == nil { 
            return 
        }
        current++ 
        ans = int(math.Max(float64(ans), float64(current)))        
        depth(root.Left) 
        depth(root.Right)
        current--        // 还原现场    
        } 
        depth(root)
    return ans
}
```


## 二叉树的最小深度

[LeetCode](https://leetcode-cn.com/problems/minimum-depth-of-binary-tree/)
给定一个二叉树，找出其最小深度。
最小深度是从根节点到最近叶子节点的最短路径上的节点数量。
说明：叶子节点是指没有子节点的节点。

### 解题思路
与前一题最大深度类似, 不同的是递归结束条件为当前节点为叶子节点, 且在存在左或右子树的情况下进行递归
```go
/**
 * Definition for a binary tree node.
 * type TreeNode struct {
 *     Val int
 *     Left *TreeNode
 *     Right *TreeNode
 * }
 */
import "math"

func minDepth(root *TreeNode) int {
    // 判断是否为空树
    if root == nil {
        return 0
    }

    ans := math.MaxInt32
    var depth func(*TreeNode, int)
    depth = func(root *TreeNode, current int) {
        // 已经保证root不为nil
        if root.Left == nil && root.Right == nil {      // 叶子节点
            ans = int(math.Min(float64(ans), float64(current)))
            return
        }
        if root.Left != nil {
            depth(root.Left, current+1)
        }
        if root.Right != nil {
            depth(root.Right, current+1)
        }
    }
    depth(root, 1)
    return ans
}
```
# 分治
## Pow(x, n)
[LeetCode](https://leetcode-cn.com/problems/powx-n/)
实现 pow(x, n) ，即计算 x 的 n 次幂函数（即，xn）。

```go
func myPow(x float64, n int) float64 {
    if n < 0 {
        return 1 / myPow(x, -n)
    }
    if n == 0 {
        return 1
    }
    sub := myPow(x, n / 2)
    if n % 2 == 0 {
        return sub * sub
    } else {
        return sub * sub * x
    }
}
```

## 括号生成
[LeetCode](https://leetcode-cn.com/problems/generate-parentheses/)
数字 n 代表生成括号的对数，请你设计一个函数，用于能够生成所有可能的并且 有效的 括号组合。
### 解题思路
这题的关键在于子问题的划分, 把原问题划分为`(a)b`来进行递归求解
```go
func generateParenthesis(n int) []string {
    if n == 0 {
        return []string{""}
    }
    ans := make([]string, 0)

    for i := 1; i <= n; i++ {
        result_a := generateParenthesis(i - 1)
        result_b := generateParenthesis(n - i)

        for _, a := range result_a {
            for _, b := range result_b {
                ans = append(ans, "(" + a + ")" + b)
            }
        }
    }
    return ans
}
```









