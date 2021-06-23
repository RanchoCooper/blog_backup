---
title: 算法进阶(02) - 栈、队列
catalog: true
date: 2021-06-22 19:26:20
subtitle: Advanced Algorithm - Stack And Queue
author: Rancho
header-img:
tags:
    - 算法
---
# 时间复杂度

## 栈/队列
* push(入栈/入队): O(1)
* pop(出栈/出队): O(1)
* access(访问栈顶/访问队头): O(1)

## 双端队列
* 队头/队尾的插入/删除/访问, 都是O(1)

## 优先队列
* 访问最值: O(1)
* 插入: 一般是O(logN), 一些高级的数据结构可以做到O(1)
* 取最值: O(logN)

# 实战

## 用切片实现栈
因为Golang中没有现成的栈可以开箱即用, 这里用切片简单实现一下, 作为模板以备不时之需

```go
import (
    "fmt"
    "sync"
)

type Stack struct {
    lock sync.RWMutex
    slice []string
}

func (s *Stack) Size() int {
    return len(s.slice)
}

func (s *Stack) IsEmpty() bool {
    return s.Size() == 0
}

func (s *Stack) Push(value string) {
    s.lock.Lock()
    defer s.lock.Unlock()

    s.slice = append(s.slice, value)
}

func (s *Stack) Pop() error {
    if s.IsEmpty() {
        return fmt.Errorf("stack is empty")
    }

    s.lock.Lock()
    defer s.lock.Unlock()

    s.slice = s.slice[:s.Size() - 1]
    return nil
}

```

## 有效的括号
[LeetCode](https://leetcode-cn.com/problems/valid-parentheses/)

给定一个只包括 '('，')'，'{'，'}'，'['，']' 的字符串 s ，判断字符串是否有效。

有效字符串需满足：

左括号必须用相同类型的右括号闭合。
左括号必须以正确的顺序闭合。

### 解题思路
利用栈的先进后出特性, 遇到左括号入栈, 遇到右括号则先与栈顶元素进行配对, 如果配对成功则出栈, 配对失败或最终栈不为空则返回false. 为了方便配对, 我们可以维护一个map, 其中key是右括号, value是左括号

```go
func isValid(s string) bool {
    if len(s) % 2 != 0 {
        // 若字符串长度为奇数, 一定配对失败
        return false
    }

    mapping := map[string]string {
        ")": "(",
        "]": "[",
        "}": "{",
    }

    var stack []string
    for _, c := range s {
        c := string(c)  // 这里需要转换下类型, 否则编译不通过
        if _, ok := mapping[c]; ok {
            // 遇到右括号, 进行配对判定
            if len(stack) > 0 {
                if stack[len(stack) - 1] == mapping[c] {
                    // 出栈
                    stack = stack[:len(stack) - 1]
                } else {
                    return false
                }
            } else {
                return false
            }
        } else {
            // 左括号, 进行入栈
            stack = append(stack, c)
        }
    }

    return len(stack) == 0
}
```



## 逆波兰表达式求值
[LeetCode](https://leetcode-cn.com/problems/evaluate-reverse-polish-notation/)

根据 逆波兰表示法，求表达式的值。

有效的算符包括 +、-、*、/ 。每个运算对象可以是整数，也可以是另一个逆波兰表达式。



说明：

整数除法只保留整数部分。
给定逆波兰表达式总是有效的。换句话说，表达式总会得出有效数值且不存在除数为 0 的情况。

### 解题思路
遇到数字则入栈, 遇到操作符先将栈顶两个元素出栈, 然后进行计算

```go
import (
    "fmt"
    "strconv"
    "sync"
)

type Stack struct {
    lock sync.RWMutex
    slice []string
}

func (s *Stack) Size() int {
    return len(s.slice)
}

func (s *Stack) IsEmpty() bool {
    return s.Size() == 0
}

func (s *Stack) Push(value string) {
    s.lock.Lock()
    defer s.lock.Unlock()

    s.slice = append(s.slice, value)
}

func (s *Stack) Pop() error {
    if s.IsEmpty() {
        return fmt.Errorf("stack is empty")
    }

    s.lock.Lock()
    defer s.lock.Unlock()

    s.slice = s.slice[:s.Size() - 1]
    return nil
}

func (s *Stack) Top() string {
    if s.IsEmpty() {
        return ""
    }
    return s.slice[len(s.slice) - 1]
}

func calculate(l, r, op string) int {
    a, _ := strconv.ParseInt(l, 10, 64)
    b, _ := strconv.ParseInt(r, 10, 64)
    switch op {
    case "+":
        return int(a + b)
    case "-":
        return int(a - b)
    case "*":
        return int(a * b)
    case "/":
        return int(a / b)
    }
    return 0
}

func evalRPN(tokens []string) int {
    var s Stack
    for _, token := range tokens {
        if token == "+" || token == "-" || token == "*" || token == "/" {
            b := s.Top()
            s.Pop()
            a := s.Top()
            s.Pop()
            s.Push(strconv.Itoa(calculate(a, b, token)))
        } else {
            s.Push(token)
        }
    }
    r, _ := strconv.ParseInt(s.Top(), 10, 64)
    return int(r)
}
```


