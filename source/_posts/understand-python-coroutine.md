---
title: 理解Python协程
catalog: true
date: 2018-12-27 22:40:02
subtitle:  Understand Python Coroutine
author: Rancho
tags:
    - Python
    - 协程
    - 编程札记
---

# 概述
在Python语言的技术书籍里，关于`协程(Coroutine)`的讨论是相当匮乏的，这也导致协程成为了最鲜为人知的Python特性。到目前为止，笔者只在《Python高级编程》和《Fluent Python》两本书中见到过详细的讨论。前者是一本年代十分久远的书，不是很建议看。

由于GIL的存在，导致Python多线程的性能甚至可能比单线程还要糟糕。于是就出现了协程，它由程序主动控制切换，没有切换线程的开销，因此执行效率极高。十分适用于IO密集型任务的场景；如果是CPU密集型，推荐多进程+协程的方式。

# 协程的演变史

- 2006年，实现了协程的底层架构，即在Python2.5中引入yield表达式，详见[PEP342](https://www.python.org/dev/peps/pep-0342/)
- 2012年，实现了改良版的生成器句法，即在Python3.3中增加yield from语法，详见[PEP380](https://www.python.org/dev/peps/pep-0380/)
- 2016年，Python3.5中增加了`await`和`async`关键字，并且`asyncio`成为标准库一员，即简化协程的使用，详见[PEP492](https://www.python.org/dev/peps/pep-0492/)

在Python官方内置asyncio之前，只有第三方库对协程的支持，比如`gevent`和`Tornado`。而Python对协程的支持，实质是通过`generator`实现的。可以说，协程在Python中的实现，就是遵循某些规则的生成器。

# 生成器简介
[PEP255](https://www.python.org/dev/peps/pep-0255/)中所定义的生成器，就是包含yield关键字的函数。当解释器执行到yield语句，就会像return语句那样返回一个值，但不同的是：此时解释器会保存对栈的引用，并在下一次通过`next()`来调用生成器时，恢复该函数栈的执行。

```python
In [1]: def simple_generator():
   ...:     yield 1
   ...:     yield 2
   ...:     yield 'rancho'
   ...:
   ...:

# 得到生成器对象
In [2]: simple_generator()
Out[2]: <generator object simple_generator at 0x110685de0>

In [3]: g = simple_generator()

In [4]: next(g)
Out[4]: 1

In [5]: next(g)
Out[5]: 2

In [6]: next(g)
Out[6]: 'rancho'

# 再次唤醒, 生成器会抛出StopIteration异常
In [7]: next(g)
---------------------------------------------------------------------------
StopIteration                             Traceback (most recent call last)
<ipython-input-7-e734f8aca5ac> in <module>()
----> 1 next(g)

StopIteration:

In [8]:
```

通过`inspect.isgeneratorfunction`，可以检查一个函数是否是生成器。阅读其源码，也可以洞察生成器函数标识的内部实现

```python
def isgeneratorfunction(object):
    """Return true if the object is a user-defined generator function.

    Generator function objects provide the same attributes as functions.
    See help(isfunction) for a list of attributes."""
    return bool((isfunction(object) or ismethod(object)) and
                object.__code__.co_flags & CO_GENERATOR)

```
另外，生成器还被定义了四种状态，它们分别是`GEN_CREATED`(创建但未激活), `GEN_RUNNING`(正被解析器执行), `GEN_SUSPENDED`(等待调用方唤醒), 以及`GEN_CLOSED`(已经结束运行).

# 生成器的进化
基于PEP255所定义的生成器, PEP342对其做了改良: 在生成器API中增加了`send()`, `throw()`以及`close()`方法。

至此，调用方可以通过`send()`方法向生成器发送数据，并且发送的数据会成为生成器函数中yield表达式的值。这是生成器向协程进化的关键一步，这样一来，生成器可以与调用方协作，产出由调用方提供的值。而throw和close方法，前者的作用是让调用方抛出异常；后者是让调用发直接终止生成器。

协程的简单使用演示
```python
In [8]: def simple_coroutine():
   ...:     print('-> coroutine started')
   ...:     x = yield
   ...:     print('-> coroutine received x: ', x)
   ...:

# 同创建生成器的方式, 调用函数得到生成器对象
In [9]: c = simple_coroutine()

In [10]: c
Out[10]: <generator object simple_coroutine at 0x1106852a0>

# 激活生成器, 此时解释器会停留在第一个yield语句, 并等待调用方传值
In [11]: next(c)
-> coroutine started

# 通过send传值, 此时协程会被唤醒, 协程函数栈继续执行, 直到碰到下一个yield表达式, 或者执行到协程定义体尾部, 抛出异常
In [12]: c.send(666)
-> coroutine received x:  666
---------------------------------------------------------------------------
StopIteration                             Traceback (most recent call last)
<ipython-input-12-21bbe31479b2> in <module>()
----> 1 c.send(666)

StopIteration:
```

同样的, 协程也有生成器的四种状态. 因为send方法的参数会成为暂停的yield表达式的值, 所以, 仅当协程处于暂停状态时, 才能调用send方法. 比如, 如果协程才被创建, 尚未激活, 便会抛错

```python
In [14]: sleep = simple_coroutine()

In [15]: sleep.send(2019)
---------------------------------------------------------------------------
TypeError                                 Traceback (most recent call last)
<ipython-input-15-b30e05632a4d> in <module>()
----> 1 sleep.send(2019)

TypeError: can't send non-None value to a just-started generator
```

如果你有兴趣仔细探究协程各个时期的状态, 可以结合`inspect.getgeneratorstate`方法, 自行探究理解协程的行为

## 预激协程
首次调用next()函数, 这一步被称为`预激(prime)`协程. 我们可以简单地通过装饰器来实现预激

```python
from functools import wraps

def coroutine(func):
    @wraps(func)
    def primer(*args, **kwargs):
        gen = func(*args, **kwargs)
        next(gen)
        return gen
    return primer
```
其实很多框架都提供了处理协程的特殊装饰器, 比如Tornado提供了`tornado.gen`, 它除了预激协程, 还会勾入事件循环. 值得注意的是, `yield from`句法本身就会自动预激, 因此和示例中的装饰器并不兼容. 而Python3.4标准库中的`asyncio.coroutine`装饰器不会预激协程, 因此兼容`yield from`

## 终止协程
因为协程的运作受调用方控制, 因此可以通过发送某个哨符值, 让协程退出. 内置的`None`和`Ellipsis`等常量就是不错的选择
另外, 底层的生成器API ---- close方法会让生成器在暂停的yield表达式处抛出`GeneratorExit`异常

    generator.close()

## 异常处理
前面提到, PEP255中, 为生成器添加了`throw`方法

    generator.throw(exc_type[, exc_value[, traceback]])

如果生成器处理了抛出的异常, 代码就会继续执行到下一个yield表达式, 而产出的值就变变成调用`generator.throw`方法得到的返回值. 如果生成器没有处理抛出的异常, 异常就会向上冒泡, 传到调用方的上下文中

## 让协程返回值
在Python3.3之前, 如果生成器返回值, 解释器就会报错. 按照PEP380中定义的方式, 获取协程的返回值需要绕个圈子

```python
# 使用前面定义的预计装饰器
In [19]: @coroutine
    ...: def simple_demo():
    ...:     while True:
    ...:         x = yield
    ...:         if x is Ellipsis:
    ...:             break
    ...:         print('receive x: ', x)
    ...:     return 'customer return value'
    ...:
    ...:

In [20]: c = simple_demo()

In [21]: c.send(3)
receive x:  3

In [22]: c.send(Ellipsis)
---------------------------------------------------------------------------
StopIteration                             Traceback (most recent call last)
<ipython-input-22-70d622b4e36a> in <module>()
----> 1 c.send(Ellipsis)

StopIteration: customer return value

In [24]: try:
    ...:     c.send(Ellipsis)
    ...: except StopIteration as exc:
    ...:     result = exc.value
    ...:     print('got result: ', result)
    ...:
got result:  None
```
不难发现, 当协程返回某个值时, 同样会抛出StopIteration异常, 并且异常的值就是协程返回的值. 对于`yield from`句法来说, 解释器不仅会捕获`StopIteration`异常, 还会把value属性的值变成yield from表达式的值

# yield from 句法
`yield from` 是全薪的语言结构, 它的作用比yield关键字多很多.

## 专用术语
- 调用方: 调用委派生成器的客户端代码
- 委派生成器: 包含`yield from <iterable>`表达式的生成器函数
- 子生成器: 委派生成器中所获取的生成器, 也就是PEP380标题中所指的`subgenerator`

![术语图解](professional-term.png)

## 简单示范

```python
In [25]: def gen():
    ...:     for c in 'rancho':
    ...:         yield c
    ...:     for i in range(5):
    ...:         yield i
    ...:

In [26]: list(gen())
Out[26]: ['r', 'a', 'n', 'c', 'h', 'o', 0, 1, 2, 3, 4]

# yield from结构改写
In [27]: def gen():
    ...:     yield from 'rancho'
    ...:     yield from range(5)
    ...:

In [28]: list(gen())
Out[28]: ['r', 'a', 'n', 'c', 'h', 'o', 0, 1, 2, 3, 4]
```
[Cook Book - 展开嵌套的序列](https://python3-cookbook.readthedocs.io/zh_CN/latest/c04/p14_flattening_nested_sequence.html)有个稍复杂的参考示例.
可能是yield from句法太过便捷, 以至于有人在[StackOverFlow](https://stackoverflow.com/questions/9708902/in-practice-what-are-the-main-uses-for-the-new-yield-from-syntax-in-python-3)上对该语法特性的用处发出了灵魂质问
实际上除了职责委派之外, yield from会在调用方和子生成器之间打开双向通道, 这样二者就可以直接发送和产出值, 并且可以直接传入异常, 而不需要位于中间的委派生成器中添加大量样板代码

在PEP380中，生成器句法被加以改动，其[proposal](https://www.python.org/dev/peps/pep-0380/#proposal)一节, 对`yield from`做了六点阐述

- 子生成器传出的值, 都直接传给委派生成器的调用方 (即客户端代码)
- 调用方通过`send()`方法传入的值, 都会直接传给子生成器. 如果发送的是`None`, 则调用子生成器的`__next__()`方法, 否则通过调用子生成器的`send()`方法, 向下传递. 如果子生成器抛出`StopIteration`异常, 那么委派生产器恢复运行, 如果子生成器抛出其他异常, 都会向上冒泡, 传递给委派生成器
- 当生成器退出时, 生成器(或者说子生成器)中的`return expr` 表达式会触发`StopIteration(expr)`异常并抛出
- `yield from` 表达式的值是子生成器终止时传给`StopIteration`异常的第一个参数

另外提及的两点特性, 与异常和终止有关

- 传给委派生成器的异常, 除了`GeneratorExit`之外都会下发给子生成器的`throw()`方法. 如果调用`throw()`方法时抛出`StopIteration`, 委派生成器恢复运行, 若抛出其他异常, 都会冒泡给委派生成器
- 如果把`GeneratorExit`异常传入委派生成器, 或者调用`close()`方法, 则会在子生成器上调用`close()`方法. 如果调用`close()`导致异常抛出, 那么异常会冒泡, 否则, 委派生成器抛出`GeneratorExit`异常.

`yield from`的语义比较难简洁地阐述出来, PEP380中附加了一段[伪代码](https://www.python.org/dev/peps/pep-0380/#formal-semantics)来演示`yield from `的行为 
下面对其做简化, 以描述`yield from`的运作方式

```python
# 假设解释器碰到下面这行yield from句法
RESULT = yield from EXPR

# 那么内部的处理逻辑如下

# EXPR是可迭代对象, 获取其迭代器
_i = iter(EXPR)

try:
    # 预激子生成器
    _y = next(_i)
except StopIteration as _e:
    _r = _e.value
else:
    # 运行这个循环时, 委派生成器会阻塞, 只作为调用方和子生成器间的通道
    while True:
        # 产出子生成器的值, 并等待调用方唤醒
        _s = yield _y
        try:
            # 下发调用方的传入值给子生成器
            _y = _i.send(_s)
        # 如果子生成器终止, 获取异常的value属性, 退出循环, 并唤醒委派生成器
        except StopIteration as _e:
            _r = _e.value
            break

RESULT = _r
```

这段简化后的代码不支持`throw()`和`close()`方法, 也没有处理传入`None`的情形, 如果想知道应对实际情况下的实现, 建议查看上面PEP的完整伪代码

# 小结
Guido大叔说生成器有三种代码编写风格: 拉取式(迭代器), 推送式(send), 以及任务式([详见](http://www.dabeaz.com/coroutines/)).
不过在实际应用中, 感觉多数项目都是采用`yield from`来驱动协程, 而不通过直接在协程上调用`send()`方法驱动, 这一切都得益于`asyncio`库的底层处理了`next()`和`send()`方法驱动, 用户代码只需使用`yield from`句法, 便可驱动协程运行

实际上自Python3.5, 提供了async/await关键字后, 基本取代了send + yield from的结构.
不过弄明白协程以及`yield from`句法, 新引入的关键字就很好理解了, 只是简化语法, 隐藏细节

# 参考文献
- 《Fluent Python》 * 主推
- 《Python高级编程》
- 《Python高手之路》
