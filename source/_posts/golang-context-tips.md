---
title: Golang Context的好坏与使用建议
catalog: true
date: 2021-08-14 00:55:07
subtitle: Golang Context Tips
author: Rancho
header-img:
tags:
    - 编程语言
    - Golang
---

`context`的设计在Golang中算是一个比较有争议的话题。`context`不是银弹，它解决了一些问题的同时，也有不少让人诟病的缺点。本文主要探讨一下`context`的优缺点以及一些使用建议。

# 缺点

由于主观上我也不是很喜欢context的设计，所以我们就从缺点先开始吧。

## 到处都是context
根据`context`使用的官方建议，`context`应当出现在函数的第一个参数上。这就直接导致了代码中到处都是`context`。作为函数的调用者，即使你不打算使用`context`的功能，你也必须传一个占位符——`context.Background()`或`context.TODO()`。这无疑是一种`code smell`，特别是对于有代码洁癖程序员来说，传递这么多无意义的参数是简直是令人无法接受的。

## context.Value——没有约束的自由是危险的

`context.Value`几乎就是一个 `map[interface{}]interface{}`：

```go
type Context interface {
    ...
    Value(key interface{}) interface{}
    ...
}
```

## 可读性很差

可读性差也是自由带来的代价，在学习阅读Go代码的时候，看到`context`是令人头疼的一件事。如果文档注释的不够清晰，你几乎无法得知`context.Value`里究竟包含什么内容，更不谈如何正确的使用这些内容了。


# 优点

## 统一了cancelation的实现方法

许多文章都说`context`解决了goroutine的cancelation问题，但实际上，我觉得cancelation的实现本身不算是一个问题，利用关闭`channel`的广播特性，实现cancelation是一件比较简单的事情，举个栗子：

```go
// Cancel触发一个取消
func Cancel(c chan struct{}) {
    select {
        case <-c: //已经取消过了, 防止重复close
        default:
            close(c)
    }
}

// DoSomething做一些耗时操作，可以被cancel取消。
func DoSomething(cancel chan struct{}, arg Arg)  {
    rs := make(chan Result)
    go func() {
        // do something
        rs <- xxx  //返回处理结果
    }()

    select {
        case <-cancel:
            log.Println("取消了")
        case result := <-rs:
            log.Println("处理完成")
    }
}
```

或者你也可以把用于取消的channel放到结构体里：

```go 
type Task struct{
    Arg Arg
    cancel chan struct{} //取消channel
}

// NewTask 根据参数新建一个Task
func NewTask(arg Arg) *Task{
    return &Task{
        Arg:arg ,
        cancel:make(chan struct{}),
    }
}

// Cancel触发一个取消
func (t *Task) Cancel() {
    select {
        case <-t.c: //已经取消过了, 防止重复close
        default:
            close(t.c)
    }
}

// DoSomething做一些耗时操作，可以被cancel取消。
func (t *Task) DoSomething() {
    rs := make(chan Result)
    go func() {
        // do something
        rs <- xxx
    }()

    select {
        case <-t.cancel:
            log.Println("取消了")
        case result := <-rs:
            log.Println("处理完成")
    }
}
// t := NewTask(arg)
// t.DoSomething()
```

可见，对cancelation的实现也是多种多样的。一千个程序员由可能写出一千种实现方式。不过幸亏有`context`统一了cancelation的实现，不然怕是每引用一个库，你都得额外学习一下它的cancelation机制了。我认为这是context最大的优点，也是最大的功劳。gopher们只要看到函数中有`context`，就知道如何取消该函数的执行。如果想要实现cancelation，就会优先考虑`context`。


## 提供了一种不那么优雅，但是有效的传值方式
`context.Value`是一把双刃剑，上文中提到了它的缺点，但只要运用得当，缺点也可以变优点。`map[interface{}]interface{}`的属性决定了它几乎能存任何内容，如果某方法需要cancelation的同时，还需要能接收调用方传递的任何数据，那`context.Value`还是十分有效的方式。如何“运用得当”请参考下面的使用建议。

# context使用建议

## 需要cancelation的时候才考虑context

`context`主要就是两大功能，cancelation和`context.Value`。如果你仅仅是需要在goroutine之间传值，请不要使用`context`。因为在Go的世界里，`context`一般默认都是能取消的，一个不能取消的`context`很容易被调用方误解。

    一个不能取消的context是没有灵魂的。

## context.Value能不用就不用

`context.Value`内容的存取应当由库的使用者来负责。如果是库内部自身的数据流转，那么请不要使用`context.Value`，因为这部分数据通常是固定的，可控的。假设某系统中的鉴权模块，需要一个字符串`token`来鉴权，对比下面两种实现方式，显然是显示将`token`作为参数传递更清晰。

```go
// 用context
func IsAdminUser(ctx context.Context) bool {
    x := token.GetToken(ctx)
    userObject := auth.AuthenticateToken(x)
    return userObject.IsAdmin() || userObject.IsRoot()
}

// 不用context
func IsAdminUser(token string, authService AuthService) int {
    userObject := authService.AuthenticateToken(token)
    return userObject.IsAdmin() || userObject.IsRoot()
}
```

所以，请忘了`request-scoped`吧，把`context.Value`想象成是`user-scoped`——让用户，也就是库的调用者来决定在`context.Value`里面放什么。

## 使用NewContext和FromContext对来存取context

不要直接使用`context.WithValue()`和`context.Value("key")`来存取数据，将`context.Value`的存取做一层封装能有效降低代码冗余，增强代码可读性同时最大限度的防止一些粗心的错误。

## 如果使用context.Value，请注释清楚


上面提到，`context.Value`可读性是十分差的，所以我们不得不用文档和注释的方式来进行弥补。至少列举所有可能的`context.Value`以及它们的get/set方法（`NewContext(),FromContext()`），尽可能的列举函数入参与`context.Value`之间的关系，给阅读或维护你代码的人多一份关爱。

## 封装以减少context.TODO()或context.Background()

对于那些提供了`context`的方法，但作为调用方我们并不使用的，还是不得不传`context.TODO()`或`context.Background()`。如果你不能忍受大量无用的`context`在代码中扩散，可以对这些方法做一层封装：

```go 
// 假设有如下查询方法，但我们几乎不使用其提供的context
func QueryContext(ctx context.Context, query string, args []NamedValue) (Rows, error) {
    ...
}

// 封装一下
func Query(query string, args []NamedValue) (Rows, error) {
    return QueryContext(context.Background(), query, args)
}
```
