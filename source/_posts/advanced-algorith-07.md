---
title: 算法进阶(07) - DFS、BFS
catalog: true
date: 2021-07-10 04:40:21
subtitle: Adcanced Algoritm - DFS & BFS
author: Rancho
header-img:
tags:
    - 算法
---

# DFS VS BFS
DFS更适合搜索树形状态空间

* 递归本身就会产生树的结构
* 可以用一个全局变量维护状态较为复杂的信息(子集/排列)
* 不需要队列, 节省空间



BFS适合求`最小代价`、`最少步数`之类的题目

* BFS是按层次序搜索, 任意时刻队列中至多只有两层



在状态空间为一般的`图`时(需要判重), DFS/BFS均可

# 实战

## 电话号码的字母组合
[LeetCode](https://leetcode-cn.com/problems/letter-combinations-of-a-phone-number/)

给定一个仅包含数字 2-9 的字符串，返回所有它能表示的字母组合。答案可以按 任意顺序 返回。

给出数字到字母的映射如下（与电话按键相同）。注意 1 不对应任何字母。

```go
func letterCombinations(digits string) []string {
    if digits == "" {
        return []string{}
    }
    
    phone := make(map[string]string)
    phone["2"] = "abc"
    phone["3"] = "def"
    phone["4"] = "ghi"
    phone["5"] = "jkl"
    phone["6"] = "mno"
    phone["7"] = "pqrs"
    phone["8"] = "tuv"
    phone["9"] = "wxyz"

    ans := make([]string, 0)
    var str string

    var dfs func(string, int)
    dfs = func(digits string, index int) {
        if index == len(digits) {
            ans = append(ans, str)
            return
        }

        for _, digit := range phone[string(digits[index])] {
            str = fmt.Sprintf("%s%s", str, string(digit))
            dfs(digits, index + 1)
            str = str[:len(str)-1]
        }
    } 

    dfs(digits, 0)
    return ans
}
```

## N 皇后
[LeetCode](https://leetcode-cn.com/problems/n-queens/)


```go
func solveNQueens(n int) [][]string {
    results := make([][]string, 0)
    ans := make([][]int, 0)
    an := make([]int, 0)
    used := make([]bool, n)
    usedIplusJ := make(map[int]bool)
    usedIminusJ := make(map[int]bool)

    var find func(int)
    find = func(row int) {
        if row == n {
            ans = append(ans, append([]int(nil), an...))   // 注意这里要复制切片
            return
        }
        for col := 0; col < n; col++ {
            if !used[col] && !usedIplusJ[row + col] && !usedIminusJ[row - col] {
                used[col] = true
                usedIplusJ[row + col] = true
                usedIminusJ[row - col] = true
                an = append(an, col)
                find(row+1)
                an = an[:len(an) - 1]
                usedIminusJ[row - col] = false
                usedIplusJ[row + col] = false
                used[col] = false
            }
        }
    }

    find(0)

    // 转换成 "..Q."的形式
    for _, each := range ans {
        result := make([]string, 0)
        for row := 0; row < n; row++ {
            s := strings.Repeat(".", n)
            b := []byte(s)
            b[each[row]] = 'Q'
            s = string(b)
            result = append(result, s)
        }
        results = append(results, result)
    }
    return results
}
```

## 岛屿数量
[LeetCode](https://leetcode-cn.com/problems/number-of-islands/)

### 解法一: DFS
```go
func numIslands(grid [][]byte) int {
    ans := 0
    m := len(grid)
    n := len(grid[0])

    // 初始化visited二维数组
    visited := make([][]bool, m)
    for i := 0; i < m; i++ {
        visited[i] = make([]bool, n)
    }

    // 方向数组
    dx := []int{-1, 1, 0, 0}
    dy := []int{0, 0, 1, -1}

    var dfs func([][]byte, int, int)
    dfs = func(grid [][]byte, x, y int) {
        // 1. 标记visited状态
        visited[x][y] = true
        // 2. 向四个方向dfs
        for i := 0; i < 4; i++ {
            nx := x + dx[i]
            ny := y + dy[i]
            // 越界情况
            if nx < 0 || ny < 0 || nx >= m || ny >= n {
                continue
            }
            if string(grid[nx][ny]) == "1" && !visited[nx][ny] {
                dfs(grid, nx, ny)
            }
        }
    }

    for i := 0; i < m; i++ {
        for j := 0; j < n; j++ {
            if string(grid[i][j]) == "1" && !visited[i][j] {
                dfs(grid, i, j)
                ans++
            }
        }
    }
    return ans
}
```

### 解法二: BFS

```go
func numIslands(grid [][]byte) int {
    ans := 0
    m := len(grid)
    n := len(grid[0])
    
    // 方向数组
    dx := []int{-1, 1, 0, 0}
    dy := []int{0, 0, 1, -1}

    visited := make([][]bool, m)
    for i := 0; i < m; i++ {
        visited[i] = make([]bool, n)
    }

    var bfs func([][]byte, int, int)
    bfs = func(grid [][]byte, x, y int) {
        visited[x][y] = true
        queue := make([][]int, 0)
        queue = append(queue, []int{x, y})

        for len(queue) != 0 {
            current := queue[0]
            queue = queue[1:]
            for i := 0; i < 4; i++ {
                nx := current[0] + dx[i]
                ny := current[1] + dy[i]

                // 越界情况
                if nx < 0 || ny < 0 || nx >= m || ny >= n {
                    continue
                }
                if string(grid[nx][ny]) == "1" && !visited[nx][ny] {
                    queue = append(queue, []int{nx, ny})
                    visited[nx][ny] = true
                }
            }
        }

    }

    for i := 0; i < m; i++ {
        for j := 0; j < n; j++ {
            if string(grid[i][j]) == "1" && !visited[i][j] {
                bfs(grid, i, j)
                ans++
            }
        }
    }

    return ans
}
```

## 最小基因变化
[LeetCode](https://leetcode-cn.com/problems/minimum-genetic-mutation/)

```go
func minMutation(start string, end string, bank []string) int {
    // 记录变化步数
    depth := make(map[string]int, 0)
    genetic := []byte{'A', 'C', 'G', 'T'}
    // 将bank转换为map, 方便比对
    bankSet := make(map[string]bool)
    for _, s := range bank {
        bankSet[s] = true
    }

    queue := make([]string, 0)
    queue = append(queue, start)
    depth[start] = 0

    for len(queue) != 0 {
        current := queue[0]
        queue = queue[1:]
        for i := 0; i < len(start); i++ {
            for j := 0; j < len(genetic); j++ {
                if current[i] == genetic[j] {
                    continue
                }
                byteNext := []byte(current)
                byteNext[i] = genetic[j]
                next := string(byteNext)
                if _, ok := bankSet[next]; !ok {
                    // next不在基因库中
                    continue
                }
                if _, ok := depth[next]; !ok {
                    depth[next] = depth[current] + 1
                    queue = append(queue, next)
                    if next == end {
                        return depth[next]
                    }
                }
            }
        }
    }
    return -1
}
```
