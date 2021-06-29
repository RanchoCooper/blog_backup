---
title:  算法进阶(03) - 前缀和、差分、双指针扫描
catalog: true
date: 2021-06-28 18:20:13
subtitle: Advanced Algorithm - Prefix-sum And Double Pointer Scanning
author: Rancho
header-img:
tags:
    - 算法
---


# 前缀和

对于一维数组A

* 前缀和数组: S[i] = S[i - 1] + A[i]
* 字段和: Sum[l, r] = S[r] - S[l - 1]
* 当A中元素都是非负数时, 前缀和数组S单调递增

## 统计「优美子数组」
[LeetCode](https://leetcode-cn.com/problems/count-number-of-nice-subarrays/)

给你一个整数数组 nums 和一个整数 k。如果某个 连续 子数组中恰好有 k 个奇数数字，我们就认为这个子数组是「优美子数组」。请返回这个数组中「优美子数组」的数目。

### 解题思路
首先, 这题和数组中元素的值无关, 我们可以先把数组进行转换: 奇数为1, 偶数为零, 然后构建前缀和数组S. 要得到`子数组中恰好有K个奇数`的数量, 也就是求S[i] - S[j] = k的数量. 所以只需要用一个计数数组统计S中每个值的个数, 枚举右端点i, 看一下等于S[i] - k的值的个数就可以了

```go
func numberOfSubarrays(nums []int, k int) int {
    // 向nums数组头部插入0, 方便下标索引   
    nums = append([]int{0}, nums...)   
    n := len(nums)   
    s := make([]int, n)   
    // 构建nums的前缀和数组, 并进行奇偶转换, 注意这里是从i = 1开始   
    for i := 1; i < n; i++ {
    	s[i] = s[i - 1] + nums[i] % 2
    }   
    // 构建前缀和值的统计数组
    count := make([]int, n)   
    for i := 0; i < n; i++ {
    	count[s[i]] += 1
    }   
    // 对于每个r(1~n), 考虑有几个l(1~r)满足s[r] - s[l - 1] = k   
    // 对于每个i(1~n), 考虑有几个j(1~r-1)满足s[i] - s[j] = k, 即   
    // 对于每个i(1~n), 考虑有几个j(1~r-1)满足s[i] - k = s[j]   
    // 也就是对于每个i, 考虑有几个s[j]等于s[i] - k   
    var ans int   
    for i := 0; i < n; i++ {
    	if s[i] - k >= 0 {
    		ans += count[s[i] - k]       
    	}   
    }   
    return ans
}
```
                                      
# 二维前缀和

对于二维数组A

* 前缀和数组S[i][j] = S[i-1][j] + S[i][j-1] - S[i-1][j-1] + A[i][j]
* 子矩阵和(以(p, q)为左上角, 以(i, j)为右下角的子矩阵的和)
  Sum(p, q, i, j) = S[i][j] - S[i][q-1] - S[p-1][j] + S[p-1][q-1]
* 复杂度: 预处理O(N2^), 询问O(1)

## 二维区域和检索 - 矩阵不可变
[LeetCode](https://leetcode-cn.com/problems/range-sum-query-2d-immutable/)

给定一个二维矩阵，计算其子矩形范围内元素的总和，该子矩阵的左上角为 (row1, col1) ，右下角为 (row2, col2) 。

# 差分

## 基础知识

对于一维数组A, 其差分数组B, 有

* B1 = A1, B[i] = A[i] - A[i-1]
* 差分数组B的前缀和数组就是原数组A

## 航班预订统计
[LeetCode](https://leetcode-cn.com/problems/corporate-flight-bookings/)

这里有 n 个航班，它们分别从 1 到 n 进行编号。有一份航班预订表 bookings ，表中第 i 条预订记录 bookings[i] = [firsti, lasti, seatsi] 
意味着在从 firsti 到 lasti （包含 firsti 和 lasti ）的 每个航班 上预订了 seatsi 个座位。

请你返回一个长度为 n 的数组 answer，其中 answer[i] 是航班 i 上预订的座位总数。

```go
func corpFlightBookings(bookings [][]int, n int) []int {   
    delta := make([]int, n + 2) // 差分数组要开0~n+1   
    for _, booking := range bookings {       
        first, last, seats := booking[0], booking[1], booking[2]       
        // 差分模板       
        delta[first] += seats
        delta[last + 1] -= seats  
    }  
    sum := make([]int, n + 1)   
    for i := 1; i <= n; i++ {      
        sum[i] = sum[i - 1] + delta[i]  
    }   
    // 将sum中元素往前挪一位   
    for i := 1; i <= n; i++ {       
        sum[i - 1] = sum[i]   
    }  
    return sum[:len(sum) - 1]
}
```

## 最大子序和

给定一个整数数组 nums ，找到一个具有最大和的连续子数组（子数组最少包含一个元素），返回其最大和。

### 解题思路

解法一: 前缀和 + 前缀最小值求出前缀和数组S, 枚举右端点i, 需要找到i之前的一个j, 使得S[i] - S[j]最大, 也就是要让S[j]最小, 再维护一个S的前缀最小值即可

```go
import "math"

func maxSubArray(nums []int) int {   
    n := len(nums)   
    sum := make([]int, n + 1)   
    // 构建前缀和   
    for i := 1; i <= n; i++ {       
        sum[i] = sum[i - 1] + nums[i - 1]  
    }  
    ans := math.MinInt32   
    preMin := sum[0]   
    for i := 1; i <= n; i++ {      
        ans = int(math.Max(float64(ans), float64(sum[i] - preMin)))      
        preMin = int(math.Min(float64(preMin), float64(sum[i])))  
    } 
    return ans
}
```

# 双指针扫描

双指针扫描一般用于解决 基于`子段`的统计问题这类题目的朴素做法都是两重循环的枚举, 枚举左端点l/右端点r. 
(时间复杂度O(N^2))优化手法都是找到枚举中的冗余部分, 并将其去除通常的优化策略有:

* 固定右端点, 看左端点的取值范围
* 移动一个端点, 看另一个端点的变化情况
	* 例如一个端点跟随另一个端点单调移动, 像一个`滑动窗口`
	* 此时可以考虑`双指针扫描`

## 两数之和 II - 输入有序数组

[LeetCode](https://leetcode-cn.com/problems/two-sum-ii-input-array-is-sorted/)

给定一个已按照 升序排列  的整数数组 numbers ，请你从数组中找出两个数满足相加之和等于目标数 target 。
函数应该以长度为 2 的整数数组的形式返回这两个数的下标值。numbers 的下标 从 1 开始计数 ，所以答案数组应当满足 1 <= answer[0] < answer[1] <= numbers.length 。
你可以假设每个输入只对应唯一的答案，而且你不可以重复使用相同的元素。

```go
func twoSum(numbers []int, target int) []int {   
    j := len(numbers) - 1   
    for i := 0; i < len(numbers); i++ {       
        // 当i增加, 若要满足numbers[i] + numbers[j] == target, j必然要减小       
        // 所以可以固定i, 令j不断减小       
        for ; i < j && numbers[i] + numbers[j] > target; j-- {                  
            	
        } 
        if i < j && numbers[i] + numbers[j] == target {           
            return []int{i + 1, j + 1}      
        }   
    }  
    return []int{}}
```

## 三数之和

给你一个包含 n 个整数的数组 nums，判断 nums 中是否存在三个元素 a，b，c ，使得 a + b + c = 0 ？请你找出所有和为 0 且不重复的三元组。
注意：答案中不可以包含重复的三元组。

### 解题思路
复用两数之和

```go
func threeSum(nums []int) [][]int {   
    ans := make([][]int, 0)       
    // 排序   
    sort.Sort(sort.IntSlice(nums))   
    for i := 0; i < len(nums); i++ {       
        // 去重       
        if i > 0 && nums[i] == nums[i - 1] {           
            continue    
        } 
        jks := twoSum(nums, i + 1, -nums[i])       
        for _, jk := range jks {      
            ans = append(ans, []int{nums[i], jk[0], jk[1]})  
        }  
    } 
    return ans
}

func twoSum(nums []int, start, target int) [][]int {   
    ans := make([][]int, 0)   
    j := len(nums) - 1  
    for i := start; i < len(nums); i++ {       
        // 去重       
        if i > start && nums[i] == nums[i - 1] {           
            continue     
        }  
        for ; i < j && nums[i] + nums[j] > target; {           
            j--    
        } 
        if i < j && nums[i] + nums[j] == target {   
            ans = append(ans, []int{nums[i], nums[j]})     
        } 
    } 
    return ans
}
```

## 盛最多水的容器

[LeetCode](https://leetcode-cn.com/problems/container-with-most-water/)

给你 n 个非负整数 a1，a2，...，an，每个数代表坐标中的一个点 (i, ai) 。在坐标内画 n 条垂直线，垂直线 i 的两个端点分别为 (i, ai) 和 (i, 0) 。
找出其中的两条线，使得它们与 x 轴共同构成的容器可以容纳最多的水。
说明：你不能倾斜容器。

### 解题思路

盛水多少是由短的那根决定的, 我们可以用i/j分别从头尾向中间夹逼

```go
import "math"

func maxArea(height []int) int {   
    ans, i, j := 0, 0, len(height) - 1   
    // i, j从两端向中间夹逼   
    for ; i < j; {       
        fi := float64(height[i])       
        fj := float64(height[j])       
        fa := float64(ans)       
        ans = int(math.Max(fa, float64(j - i) * math.Min(fi, fj)))       
        if fi < fj {         
            i++    
        } else {    
            j--     
        }
    } 
    return ans
}
```
