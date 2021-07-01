---
title: 算法进阶(04) - 哈希、集合、映射
catalog: true
date: 2021-06-30 17:33:32
subtitle: Advanced Algorithm - Hash/Set And Map
author: Rancho
header-img:
tags:
    - 算法
---
# 基础知识

## 哈希表的原理

哈希表(Hash Table)又称散列表, 是一种可以通过`关键码`(key)直接进行访问的数据结构. 它由两部分组成

* 一个数据结构, 通常是链表、数组
* Hash函数, 输入`关键码`, 返回对应数据的索引

对外表现可以通过关键码直接访问, 如

    hash_table[key] = value

实际上是对key进行了一次哈希运算之后, 从数据结构中定位到value    

    data_structure[hash(key)] = value

所以设计哈希表的关键在于设计一个理想的Hash函数, 它能把复杂信息映射到较小的值域内, 作为索引. 

## 哈希碰撞

哈希碰撞是指两个不同的key被计算出同样的哈希值, 好的哈希函数可以减少碰撞发生的几率, 让数据尽可能均匀地分布开散列是最常见的碰撞解决方案

# 实战

## 两数之和
[LeetCode](https://leetcode-cn.com/problems/two-sum/description/)

给定一个整数数组 nums 和一个整数目标值 target，请你在该数组中找出 和为目标值 target  的那 两个 整数，并返回它们的数组下标。

你可以假设每种输入只会对应一个答案。但是，数组中同一个元素在答案里不能重复出现。

你可以按任意顺序返回答案。

```go
func twoSum(nums []int, target int) []int {
    m := make(map[int]int)

    for i := 0; i < len(nums); i++ {
        another := target - nums[i]
        if _, ok := m[another]; ok {
            return []int{m[another], i}
        }
        m[nums[i]] = i
    }
    return []int{}
}
```


## 模拟行走机器人
[LeetCode](https://leetcode-cn.com/problems/walking-robot-simulation/)

机器人在一个无限大小的 XY 网格平面上行走，从点 (0, 0) 处开始出发，面向北方。该机器人可以接收以下三种类型的命令 commands ：

-2 ：向左转 90 度
-1 ：向右转 90 度
1 <= x <= 9 ：向前移动 x 个单位长度
在网格上有一些格子被视为障碍物 obstacles 。第 i 个障碍物位于网格点  obstacles[i] = (xi, yi) 。

机器人无法走到障碍物上，它将会停留在障碍物的前一个网格方块上，但仍然可以继续尝试进行该路线的其余部分。

返回从原点到机器人所有经过的路径点（坐标为整数）的最大欧式距离的平方。（即，如果距离为 5 ，则返回 25 ）


注意：

北表示 +Y 方向。
东表示 +X 方向。
南表示 -Y 方向。
西表示 -X 方向。

### 解题思路
解题关键在于利用方向数组来进行方向转换

## 字母异位词分组
[LeetCode](https://leetcode-cn.com/problems/group-anagrams/)

给定一个字符串数组，将字母异位词组合在一起。字母异位词指字母相同，但排列不同的字符串。

示例:

输入: ["eat", "tea", "tan", "ate", "nat", "bat"]
输出:
[
["ate","eat","tea"],
["nat","tan"],
["bat"]
]
说明：

所有输入均为小写字母。
不考虑答案输出的顺序。

### 解题思路
遍历输入, 对每个字符串的副本进行排序, 并将排序后的字符串作为key, 原字符串作为value保存到一个map中

注意, Go语言对字符串的排序要特殊处理

```go
import "sort"

type SortRunes []rune

func (s SortRunes) Len() int {
    return len(s)
}

func (s SortRunes) Less(i, j int) bool {
    return s[i] < s[j]
}

func (s SortRunes) Swap(i, j int) {
    s[i], s[j] = s[j], s[i]
}

func groupAnagrams(strs []string) [][]string {
    mapping, ans := map[string][]string{}, [][]string{}

    for _, str := range strs {
        sorted := []rune(str)
        sort.Sort(SortRunes(sorted))
        value := mapping[string(sorted)]
        value = append(value, str)
        mapping[string(sorted)] = value
    }

    for _, v := range mapping {
        ans = append(ans, v)
    }
    return ans
}
```

## 串联所有单词的子串
[LeetCode](https://leetcode-cn.com/problems/substring-with-concatenation-of-all-words/)

给定一个字符串 s 和一些 长度相同 的单词 words 。找出 s 中恰好可以由 words 中所有单词串联形成的子串的起始位置。

注意子串要与 words 中的单词完全匹配，中间不能有其他字符 ，但不需要考虑 words 中单词串联的顺序。

### 解题思路
这题的解法偏暴力, 先将所有的words转成映射, 然后以step为步进依次查看当前的单词是否在m中

做完提交后才发现这题不讲武德, words里可能会有重复的单词. 所以在每次迭代i时都重新去构建映射m. 如果题目中的words不包含重复单词, 这种解法会快很多

```go
func findSubstring(s string, words []string) []int {
    step := len(words[0])

    ans := []int{}
    for i := 0; i < len(s) - len(words) * step + 1; i++ {
        flag := true
        m := map[string]int{}
        for _, word := range words {
            m[word]++
        }
        for j := 0; j < len(words); j++ {
            key := s[i+j*step: i+(j+1)*step]
            if _, ok := m[key]; !ok {
                flag = false
                break
            }
            m[key]--
            if m[key] < 0 {
                flag = false
                break
            }
        }
        if flag {
            ans = append(ans, i)
        }
    }
    return ans
}
```

## LRU 缓存机制
[LeetCode](https://leetcode-cn.com/problems/lru-cache/)

运用你所掌握的数据结构，设计和实现一个  LRU (最近最少使用) 缓存机制 。
实现 LRUCache 类：

LRUCache(int capacity) 以正整数作为容量 capacity 初始化 LRU 缓存
int get(int key) 如果关键字 key 存在于缓存中，则返回关键字的值，否则返回 -1 。
void put(int key, int value) 如果关键字已经存在，则变更其数据值；如果关键字不存在，则插入该组「关键字-值」。当缓存容量达到上限时，它应该在写入新数据之前删除最久未使用的数据值，从而为新的数据值留出空间。

```go
type LinkedNode struct {
    key, value int
    prev, next *LinkedNode          // 双向链表
}

type LRUCache struct {
    size, capacity int
    cache map[int]*LinkedNode
    head, tail *LinkedNode          // 保护节点
}


func Constructor(capacity int) LRUCache {
    l := LRUCache{
        size: 0,
        capacity: capacity,
        cache: make(map[int]*LinkedNode, 0),
        head: &LinkedNode{},
        tail: &LinkedNode{},
    }
    // 设置保护节点
    l.head.next = l.tail
    l.tail.prev = l.head
    return l
}

func (this *LRUCache) RemoveNode(node *LinkedNode) {
    node.prev.next = node.next
    node.next.prev = node.prev
}

func (this *LRUCache) InsertFront(node *LinkedNode) {
    node.prev = this.head
    node.next = this.head.next
    this.head.next.prev = node
    this.head.next = node
}

func (this *LRUCache) Get(key int) int {
    if _, ok := this.cache[key]; !ok {
        return -1
    }
    this.RemoveNode(this.cache[key])
    this.InsertFront(this.cache[key])
    return this.cache[key].value
}


func (this *LRUCache) Put(key int, value int)  {
    if _, ok := this.cache[key]; ok {
        this.cache[key].value = value
        this.RemoveNode(this.cache[key])
        this.InsertFront(this.cache[key])
    } else {
        node := &LinkedNode{
            key: key,
            value: value,
        }
        this.cache[key] = node
        this.InsertFront(node)
        this.size++

        if this.size > this.capacity {
            delete(this.cache, this.tail.prev.key)
            this.RemoveNode(this.tail.prev)
            this.size--
        }
    }
}

```
