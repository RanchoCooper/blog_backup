---
title: 固定类型数据序列 - array
catalog: true
date: 2017-10-01 15:41:27
subtitle: Sequence of Fixed-Type Data
header-img:
tags:
    - 编程札记
    - Python标准库
---

`array`模块定义了一个序列数据结构，看起来它和list非常相似，只不过所有成员都必须是相同的基本类型。

# 初始化
array实例化时可以提供一个参数来描述允许哪种数据类型，还有一个初始的数据序列存储在数组中。
下面的例子将数组配置为`Unicode character`序列，并用一个简单的字符串来初始化。

```python
import array
import binascii

s = 'this is the array.'
a = array.array('u', s)

print('as string: ', s)
print('as array: ', a)
print('as hex: ', binascii.hexlify(a))
```

# 处理数组
类似其他的Python序列，可以采用同样的方式扩展和处理array。支持的操作包括末尾插入元素，分片以及迭代。

```python
import array
import pprint

a = array.array('i', range(5))
print('init: ', a)

a.extend(range(5))
print('extended: ', a)

print('slice: ', a[2:5])

print('iterator: ', list(enumerate(a)))
```

# 数组与文件
可以使用高效的内置文件读写方法将数组呢哦荣写入文件或者从文件读入数组。
下面的例子展示了直接从二进制文件读取原始数据，将它读入一个新的数组，并把字节转换为适当的类型。

```python
import array
import binascii
import tempfile

a = array.array('i', range(5))
print('A1: ', a)

# write the array of numbers to a temporary file
output = tempfile.NamedTemporaryFile()

a.tofile(output.file)       # must pass an *actual* file
output.flush()

# read the raw data
with open(output.name, 'rb') as input:
    raw_data = input.read()
    print('Raw contents: ', binascii.hexlify(raw_data))

    #read the data into an array
    input.seek(0)
    a2 = array.array('i')
    a2.fromfile(input, len(a))
    print('A2: ', a2)
```

# 候选字节顺序
如果数组中的数据没有采用固有的字节顺序，或者在发送到一个采用不同字节顺序的系统(这通常发生在网络传输中)之前需要交换顺序，可以通过`byteswap`方法来交换C数组中的字节顺序，比在Python中循环处理数据要高效很多。
