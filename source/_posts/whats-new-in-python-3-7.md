---
title: Python3.7新特性概览
catalog: true
date: 2018-06-29 10:44:25
subtitle: What's New In Python3.7
author: Rancho
header-img:
tags:
    - 语言特性
    - Python动态
---

Python官方于6.27发布了V3.7.0的更新说明，包含很多新特性和优化。
另外值得注意的是，此次更新**不完全向后兼容**。

## 汇总

- 新语法特性：
    * [PEP 563](https://docs.python.org/3/whatsnew/3.7.html#whatsnew37-pep563)推迟加载类型注释，依赖`from __future__ import annotations`·

- 向后不兼容的语法更改：
    * `async` 和 `await` 现在是保留关键字，可能部分三方库会受到影响

- 新的库模块：
    * contextvars: PEP 567 – Context Variables
    * dataclasses: PEP 557 – Data Classes
    * importlib.resources

- 新的built-in特性
    * PEP 553, 新函数 breakpoint()，免去了`pdb`调试手动设置断点的麻烦

- Python数据模型改进
    * PEP 562, customization of access to module attributes.（访问模块属性，可以定制了）
    * PEP 560, core support for typing module and generic types.
    * the insertion-order preservation nature of dict objects has been declared to be an official part of the Python language spec.

- 标准库中的重要改进
    * The asyncio module has received new features, significant usability and performance improvements. （asyncio 模块已经获得了新功能，显着的可用性和性能改进。）
    * The time module gained support for functions with nanosecond resolution. （time 模块获得了对纳秒级分辨率功能的支持）

- CPython 实现的改进
    * Avoiding the use of ASCII as a default text encoding: 取消 ASCII 作为默认文本编码
    * PEP 538, legacy C locale coercion
    * PEP 540, forced UTF-8 runtime mode
    * PEP 552, deterministic .pycs
    * the new development runtime mode
    * PEP 565, improved DeprecationWarning handling

- C API改进
    * PEP 539, new C API for thread-local storage

- 文档改进
    * Python官方文档新增日韩法三种语言版本


## 小结
3.7最大的问题来自把`async`和`await`作为了保留字，这就意味着至少`Cellery`短期是不能迁移升级的，因为里面有不少保留字的使用。

抱着尝新的心态，升级了本机Python，基本无痛切换。

关于多版本管理，可考虑`pipenv` + `pyenv`。
后者主要提供Python版本管理的解决方案，当使用`pipenv --python special_version`来创建虚拟环境时，`pipenv`会优先从`pyenv`的`~/.pyenv/versions`目录下查找匹配的版本。
