---
title: Python3.7新特性概览
catalog: true
date: 2018-06-29 10:44:25
subtitle: What's New In Python3.7
author: Rancho
tags:
    - 语言特性
    - Python
---

Python官方于6.27发布了V3.7.0的更新说明，包含很多新特性和优化。
另外值得注意的是，此次更新**不完全向后兼容**。

[官方文档](https://docs.python.org/3/whatsnew/3.7.html)


## 主要亮点

- 新语法特性：
    * [PEP 563](https://docs.python.org/3/whatsnew/3.7.html#whatsnew37-pep563)延迟加载类型注解，依赖`from __future__ import annotations`·

- 向后不兼容的语法更改：
    * `async` 和 `await` 现在是保留关键字，可能部分三方库会受到影响。

- 新的库模块：
    * contextvars: PEP 567 – Context Variables
    * dataclasses: PEP 557 – Data Classes
    * importlib.resources

- 新的built-in特性
    * [PEP 553](https://www.python.org/dev/peps/pep-0553/), 添加新的内置函数 `breakpoint()`，免去了以往使用`pdb`手动设置断点的麻烦。

- Python数据模型改进
    * [PEP 562](https://www.python.org/dev/peps/pep-0562/), 通过定义`__getattr__`和`__dir__`属性，来定制访问模块访问。
    * [PEP 560](https://www.python.org/dev/peps/pep-0560/), core support for typing module and generic types.
    * 保留字典的插入顺序已成为官方语言的规范。

- 标准库中的重要改进
    * `asyncio` 模块添加了新特性，并做了性能优化。
    * `time` 模块做到纳秒级别的支持。

- CPython 实现的改进
    * 取消 ASCII 作为默认文本编码(强制`UTF-8` 的运行时编码环境)
        * PEP 538, legacy C locale coercion
        * PEP 540, forced UTF-8 runtime mode
    * PEP 552, deterministic .pycs
    * the new development runtime mode
    * PEP 565, 优化`DeprecationWarning` 处理方式。交互环境或脚本执行方式会直接出发警告，而在框架或库导入时，则会静默处理这些警告。另外提供了三种警告行为供开发者使用：`FutureWarning`, `DeprecationWarning`, `PendingDeprecationWarning`

- C API改进
    * PEP 539, new C API for thread-local storage

- 文档改进
    * Python官方文档新增日韩法三种语言版本

另外，3.7版本也着重对性能做了优化，更多细节可见[Optimizations](https://docs.python.org/3/whatsnew/3.7.html#whatsnew37-perf)


## 特性详解

### 延迟加载类型注释

类型注解是Python3.x中引入的新特性，但目前的类型注解特性存在两个明显的可用性问题：
- 类型注解只能使用当前作用域内的已有类型，换言之，它不支持类型的前向引用。
- 类型注释的实现方式影响了Python的启动时间。

通过延迟加载类型注释，可以完美解决这两个问题。编译器不是在类型注解的定义处马上编译并执行代码，而是将注解以字符串的形式存储起来(可以理解为是存在表达式对应的AST结构中)。在需要时，可以通过`typing.get_type_hints()` 在运行时解析注解。由于通常注解都不是运行时所需，所以这种方案通过廉价的存储开销换来启动速度的提升。

当然，延迟加载最直观的体现是在下面这种情况

```python
import typing
from __future__ import annotations

class Relation:
    @classmethod
    def contact(cls, someone: People, anotherone: People) -> Relation:
        pass

print(Relation.contact.__annotations__)             # it works though haven't define `People`
print(typing.get_type_hints(Relation.contact))      # trigger error cause evaluate type hint at runtime


class People:
    pass

print(typing.get_type_hints(Relation.contact))      # it works too
```

在实际使用时需要通过`__future__` 手动导入延迟加载特性，未来在Python4.0中将会是默认行为。

### 内置breakpoint()

内置的`breakpoint()` 会调用`sys.brewkpointhook()`。
默认情况下后者会导入`pdb` 然后调用`pdb.set_trace()`。通过重新绑定`sys.breakpointhook`，你也可以定制使用其他调试器。实际上绑到任何`callable` 对象都是可以的。另外，也可以通过配置`PYTHONBREAKPOINT` 环境变量来设定调试器，当将其设为`0` 则表示禁用内置的`breakpoint()`

### 定制访问模块属性

在模块级别上支持定义`__getattr__()`与`__dir__()`，用武之地的可能就是模块属性的弃用警告和延迟加载。

- 模块或库升级的弃用警告

```python
import warnings

warnings.filterwarnings('default')  # Python3.2开始会默认隐藏DeprecationWarning


def new_function(*args, **kwargs):
    print('[actually] using new function')


deprecated_map = {
    'old_function': new_function
}


def __getattr__(name):
    if name in deprecated_map:
        switch_to = deprecated_map[name]
        warnings.warn(f"{name} is deprecated. Switch to {__name__}.{switch_to.__name__}", DeprecationWarning)
        return switch_to
    raise AttributeError(f"module {__name__} has no attribute {name}")
```

效果：
```bash
>>> from lib import old_function
/Users/rancho/blog/lib.py:22: DeprecationWarning: old_function is deprecated. Switch to lib.new_function
>>> old_function
<function lib.new_function(*args, **args)>
>>> old_function(1)
[actually] using new function
```

- 惰性加载

> 惰性加载有点像linux内核中的写时拷贝。写时拷贝是当fork出的进程发生了数据变更时，才从父进程拷贝数据。类似的，惰性加载就是当导入的对象实际发生调用时才会将其加载到当前内存空间中。这种做法可以避免因过早地导入过大的数据对象但实际上尚未使用而导致的空间浪费。简言之，就是从立即导入变成了用时导入。

按照以往的模块导入方式，必须要等模块里的相关属性/函数/类都准备妥当了，才完成模块导入的工作。惰性加载在模块导入阶段能大大提升效率

PEP中演示的例子

```python
# lib/__init__.py
import importlib

__all__ = ['submod']


def __getattr__(name):
    if name in __all__:
        return importlib.import_module("." + name, __name__)
    raise AttributeError(f"module {__name__!r} has no attribute {name!r}")

# lib/submod.py
print('submodule loaded')


class A:
    pass


# main.py
import lib


def trigger():
    lib.submod.A
```

可以发现，只有当main.trigger执行时，才会输出`submodule loaded`


### 弃用警告的展示
自Python3.2起，默认不显示弃用警告。此次更新中，当触发它们的代码直接在`__main__` 模块中运行便会显示。默认情况下，导入应用程序，库和框架时若触发弃用警告，仍然会被隐藏。

另外，标准库现在允许开发人员使用以下三种警告行为：
- `FutureWarning`: 默认情况下始终显示，如果你期望警告能被用户看见，那就用这个。
- `DeprecationWarning`: 在运行单测以及`__main__`中运行会显示，比较适合开发场景。
- `PendingDeprecationWarning`: 只在运行单测时显示。


### 充分支持原生类型
起初按照PEP484的设计，核心CPython解释器是不会随Python社区和语言的演进而变更。随着类型注解和`typing`模块在社区中的广泛使用，因为这一限制最终被移除。此次引入两个特殊方法：`__class_getitem__()`和`__mro_entries__()`。这两个方法主要在`typing`中被使用，结果是相关操作的性能得到了七倍的提升。


## 小结
3.7最大的问题来自把`async`和`await`作为了保留字，不少框架/库会面临兼容问题。

抱着尝新的心态，升级了本机Python，基本无痛切换。

关于多版本管理，可考虑`pipenv` + `pyenv`。前者用于管理虚拟环境，后者主要提供Python版本管理的解决方案，当使用`pipenv --python special_version`来创建虚拟环境时，`pipenv`会优先从`pyenv`的`~/.pyenv/versions`目录下查找匹配的版本。
