---
title: 算法进阶 - 数组、链表
catalog: true
date: 2021-06-20 04:58:29
subtitle: advanced-algorithm-array-and-list
author: Rancho
header-img:
tags:
    - 算法
---

# 数组

## 合并两个有序数组
[LeetCode](https://leetcode-cn.com/problems/merge-sorted-array/)

给你两个有序整数数组`nums1` 和 `nums2`，请你将 `nums2` 合并到`nums1`中，使 `nums1` 成为一个有序数组。

初始化`nums1` 和 `nums2` 的元素数量分别为`m` 和 `n` 。你可以假设`nums1` 的空间大小等于`m + n`，这样它就有足够的空间保存来自 `nums2\ 的元素。

### 解题思路
nums1有足够空间容纳所有数组元素, 用k从m+n-1处开始向前遍历nums1, 然后分别用i, j从后往前遍历两个数组, 并将大的元素填入k的位置, 时间复杂度为O(n)

```go
func merge(nums1 []int, m int, nums2 []int, n int)  {
    i := m -1
    j := n - 1
    for k := m + n - 1; k >=0; k-- {
        if j < 0 || (i > 0 && nums1[i] >= nums2[j]) {
            nums1[k] = nums1[i]     
            i--
        } else {
            nums1[k] = nums1[j]
            j--
        }
    }
}
```

## 删除有序数组中的重复项
[LeetCode](https://leetcode-cn.com/problems/remove-duplicates-from-sorted-array/)

给你一个有序数组 `nums` ，请你 `原地` 删除重复出现的元素，使每个元素 只出现一次 ，返回删除后数组的新长度。

不要使用额外的数组空间，你必须在 `原地` 修改输入数组 并在使用 O(1) 额外空间的条件下完成。

### 解题思路
遍历数组并判断i与i-1处元素是否相等, 并用n标记下标, 如果元素不相等, 将i处元素填入n

```go
func removeDuplicates(nums []int) int {
    n := 0
    for i := 0; i < len(nums); i++ {
        if i == 0 || nums[i - 1] != nums[i] {
            nums[n] = nums[i]
            n++
        }
    }
}
```


## 移动零
[LeetCode](https://leetcode-cn.com/problems/move-zeroes/)

给定一个数组 nums，编写一个函数将所有 0 移动到数组的末尾，同时保持非零元素的相对顺序。

### 解题思路
同上题, 遍历数组并判断i处元素是否为0, 并用n来标记下标, 如果不为0则将该元素填入n处

```go
func moveZeroes(nums []int)  {
    n := 0
    for i := 0; i < len(nums); i++ {
        if nums[i] != 0 {
            nums[n] = nums[i]
            n++
        }
    }

    // 将剩余的元素全部置0
    for i := n; i < len(nums); i++ {
        nums[i] = 0
    }
}
```

# 单链表

## 反转链表
[LeetCode](https://leetcode-cn.com/problems/reverse-linked-list/)

给你单链表的头节点 head ，请你反转链表，并返回反转后的链表。

### 解题思路
遍历链表, 并改变指向, 注意操作顺序

```go
/**
 * Definition for singly-linked list.
 * type ListNode struct {
 *     Val int
 *     Next *ListNode
 * }
 */
func reverseList(head *ListNode) *ListNode {
    var last *ListNode
    for head != nil {
        next := head.Next
        head.next = last
        last = head
        head = next
    }
    return last
}
```

## 环形链表
[LeetCode](https://leetcode-cn.com/problems/linked-list-cycle/)

给定一个链表，判断链表中是否有环。

如果链表中有某个节点，可以通过连续跟踪 next 指针再次到达，则链表中存在环。 为了表示给定链表中的环，我们使用整数 pos 来表示链表尾连接到链表中的位置（索引从 0 开始）。 如果 pos 是 -1，则在该链表中没有环。注意：pos 不作为参数进行传递，仅仅是为了标识链表的实际情况。

如果链表中存在环，则返回 true 。 否则，返回 false 。

### 解题思路
通过快慢指针遍历链表, 如果两个指针相遇, 则有环

```go
/**
 * Definition for singly-linked list.
 * type ListNode struct {
 *     Val int
 *     Next *ListNode
 * }
 */
func hasCycle(head *ListNode) bool {
    fast := head
    for fast != nil && fast.Next != nil {
        fast = fast.Next.Next
        head = head.Next
        if fast == head {
            return true
        }
    }     return false
}
```




