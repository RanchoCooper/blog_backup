---
title: python 单例模式
catalog: true
date: 2018-06-13 10:36:15
subtitle: Singleton in Python
author: Rancho
header-img: header.jpeg
tags:
    - 编程札记
    - 设计模式
    - Python
---

# 单例模式

单例模式(Singleton Pattern)是最常见的一种软件设计模式, 该模式的主要目的是`保证一个类在程序生命周期内只有一个实例存在`.

比如, 某个服务器程序的配置信息存放在一个文件中, 客户端通过一个`AppConfig`的类来读取配置文件的信息. 如果在程序运行期间, 有很多地方都需要使用配置文件的内容, 也就是说, 很多地方都需要创建`AppConfig`对象的实例, 这就导致系统中存在多个`AppConfig`的实例对象, 而这样会严重浪费内存资源, 尤其是在配置文件内容很多的情况下. 事实上, 类似`AppConfig`这样的类, 我们希望在程序运行期间只存在一个实例对象. 

在Python中, 有以下方法可以实现单例模式:

- 使用模块
- 使用装饰器
- 使用类
- 基于`__new__`
- 基于`metaclass`


# 实现方式

## 使用模块
Python的模块本身就是天然的单例模式, 因为在模块第一次导入时, 就会生成`.pyc`文件. 当第二次导入时, 就会直接加载`.pyc`, 而不是再次加载模块代码. 因此, 我们只需要把相关的函数和数据定义在一个模块中, 就可以得到一个单例对象了. 

```python
# singleton_module.py

class Singleton(object):

    def func(self):
        pass

singleton = Singleton()
```

在模块中将实例化对象存入变量, 使用时在其他文件中导入即可. 这种方式简单易实现. 得于斯者毁于斯, 因为这种实现方式依赖模块的一次性导入, 如果通过`imp.reload`重新加载模块, 就会重新实例化对象, 因此并非可靠的单例模式最佳实践.

```python
from singleton_module import singleton
```

##  使用装饰器
装饰器可以动态修改类或函数的功能, 我们通过创建类装饰器来管理实例化对象, 使其只能生成唯一实例, 同样可以实现单例模式.

```python
from functools import wraps

def singleton(cls):
    instances = {}

    @wraps(cls)
    def getinstance(*args, **kwargs):
        if cls not in instances:
            instances[cls] = cls(*args, **kwargs)
        return instances[cls]

    return getinstance


@singleton
class MySingletonClass(object):
    pass
```

我们定义一个装饰器, 它在内部维护`类: 实例`字典, 如果键`cls`不存在, 则进行实例化, 并将创建的对象作为值存入该字典, 否则直接返回`instances[cls]`.
Note: **不支持多线程**

## 使用类

```python
class Singleton(object):
    def __init__(self):
        pass

    @classmethod
    def instance(cls, *args, **kwargs):
        if not hasattr(Singleton, "_instance"):
            Singleton._instance = Singleton(*args, **kwargs)
        return Singleton._instance
```

使用多线程, 探究其是否是线程安全的

```python
import threading

def task():
    obj = Singleton.instance()
    print(obj)

for i in range(10):
    t = threading.Thread(target=task)
    t.start()
```

实测结果
```
<__main__.Singleton object at 0x10b0d8f98>
<__main__.Singleton object at 0x10b0d8f98>
<__main__.Singleton object at 0x10b0d8f98>
<__main__.Singleton object at 0x10b0d8f98>
<__main__.Singleton object at 0x10b0d8f98>
<__main__.Singleton object at 0x10b0d8f98>
<__main__.Singleton object at 0x10b0d8f98>
<__main__.Singleton object at 0x10b0d8f98>
<__main__.Singleton object at 0x10b0d8f98>
<__main__.Singleton object at 0x10b0d8f98>
```

乍一看是线程安全的, 实则不然, 因为执行速度过快, 第一个线程其他线程启动前就已经完成了实例的创建工作, 所以其他线程会复用这个实例. 但如果我们在类的`__init__`方法中加入一些IO操作, 这样一来, 在时序上, 第一个对象的创建要比其他线程的启动要晚, 这样其他线程就会去创建自己的实例对象.

```python
def __init__(self):
    from time import sleep
    sleep(1)
```

再次执行代码, 会发现每个线程独会分别创建自己的实例. 由此可见, 这种单例模式的实现方式无法支撑多线程.
对于线程安全问题, 解决办法就是**加锁**
未加锁的代码并发执行, 而加锁代码串行执行, 虽然牺牲了运行速度, 但保证了数据安全.

```python
from time import sleep
import threading

class Singleton(object):
    _instance_lock = threading.Lock()

    def __init__(self):
        sleep(1)

    @classmethod
    def instance(cls, *args, **kwargs):
        with Singleton._instance_lock:
            if not hasattr(Singleton, '_instance'):
                cls._instance = cls(*args, **kwargs)
        return cls._instance
```

实测结果
```
<__main__.Singleton object at 0x10ae53e48>
<__main__.Singleton object at 0x10ae53e48>
<__main__.Singleton object at 0x10ae53e48>
<__main__.Singleton object at 0x10ae53e48>
<__main__.Singleton object at 0x10ae53e48>
<__main__.Singleton object at 0x10ae53e48>
<__main__.Singleton object at 0x10ae53e48>
<__main__.Singleton object at 0x10ae53e48>
<__main__.Singleton object at 0x10ae53e48>
```

这样一来, 我们就实现了线程安全的单例模式.

### 改进装饰器
同样, 我们可以对装饰器方案也加上线程锁, 来实现线程安全

```python
from functools import wraps
import threading

def singleton(cls):
    instances = {}
    instances_lock = {}

    @wraps(cls)
    def getinstance(*args, **kwargs):
        if cls not in instances_lock:
            instances_lock[cls] = threading.Lock()

        with instances_lock[cls]:
            if cls not in instances:
                instances[cls] = cls(*args, **kwargs)
        return instances[cls]

    return getinstance

@singleton
class MySingletonClass(object):
    pass
```

通过线程锁, 我们就得到了两种线程安全的单例模式实现方案.
横向对比, 如果我们需要定义多个单例类, 装饰器方案可以最大程度地复用代码.

## 基于`__new__`
在Python中实例化一个对象时, 实际上是执行类的`__new__`方法, 得到实例对象后, 才会执行`__init__`方法进行对象初始化
而在上面的类实现方式中, 要求我们必须通过`obj = Singleton.instance()`来实例化对象, 所以更完美的方案是重载`__new__`方法.

```python
import threading


class Singleton(object):
    _instance_lock = threading.Lock()

    def __new__(cls, *args, **kwargs):
        with cls._instance_lock:
            if not hasattr(cls, '_instance'):
                cls._instance = super(Singleton, cls).__new__(cls)
        return cls._instance

    def __init__(self):
        from time import sleep
        sleep(1)

obj1 = Singleton()
obj2 = Singleton()

def task():
    print(Singleton())

for i in range(10):
    t = threading.Thread(target=task)
    t.start()
```

## 基于metaclass
> 元类(参考: [深刻理解Python中的元类](http://blog.jobbole.com/21351/))是用于创建类对象的类, 类对象创建实例对象时, 一定会调用`__call__`方法. 因此在调用`__call__`时, 保证始终只创建一个实例即可

```python
import threading


class Singleton(type):
    _instance_lock = threading.Lock()

    def __call__(cls, *args, **kwargs):
        print(cls)
        if not hasattr(cls, '_instance'):
            cls._instance = super(Singleton, cls).__call__(*args, **kwargs)
        return cls._instance


class MySingleton(object):
    __metaclass__ = Singleton

c = MySingleton()
```

# 后记
昨天面了小红书基础框架部, 面试中被问到单例模式的实现, 虽然给出了主要的三种实现方案, 但面试官问及线程安全时, 脑子一片空白, 特此补充下这方面的知识.

本篇主要参考资料:

- [Python中单例模式的几种实现方式及优化](https://www.cnblogs.com/huchong/p/8244279.html)
- [Python创建单例模式的三种方式](http://python.jobbole.com/87791/)
