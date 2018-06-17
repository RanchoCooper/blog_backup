---
title: 容器数据类型 - collections
catalog: true
date: 2017-10-01 14:07:47
subtitle: Cotainer Data Types
header-img:
tags:
    - 编程札记
    - Python标准库
---

`collections`模块包含多种数据结构的实现，扩展了其他模块中的相应结构。

- Counter：计数器
- defaultdict：带有默认值的字典。
- OrderedDict：记忆元素顺序的有序字典
- deque：双端队列，允许从任意一端增加或删除元素。
- namedtuple：生成可以使用名字来访问元素内容的tuple子类

# Counter
Counter作为一个容器，可以跟踪相同的值增加了多少次。Counter支持3种形式的初始化：调用Counter构造函数时提供元素序列或者一个包含键和技术的字典，还可以使用关键字参数将字符串名映射到计数。(默认行为会按照key来排序)

```python
from collections import Counter

Counter(['a', 'b', 'c', 'a', 'c', 'c'])
Counter({'a': 5, 'b': 1, 'c': 3})
Counter(a=2, b=1, c=3)

# got the same results
# Counter({'a': 2, 'b': 1, 'c': 3})
```

如果不提供任何参数，可以构造一个空的Counter，然后通过`update`方法填充。

```python
from collections import Counter

c = Counter()
# Counter()

c.update('abbcccdddd')
# Counter({'a': 1, 'b':2, 'c': 3, 'd':4})

c.update(a=2, d=5, e=1)
# Counter({'a': 3, 'b':2, 'c': 3, 'd':9, 'e': 1})
```

一旦填充了Counter，可以通过字典API来获取它的值。

```python
from collections importCounter

c = Counter('abcdaab')

for letter in 'abcde':
    print('{0}: {1}'.format(letter, c[letter]))
```
对于未知的元素，Counter不会产生KeyError。如果在输入中没有找到某个值，其计数为0。
`elements`方法返回一个迭代器，返回Counter记录的所有元素，不过不能保证元素的顺序不变。
`most_common`方法生成一个序列，返回计数最大的N项元素，如果不提供参数，则会按频度排序返回所有列表。

```python
for letter, count in c.most_common(3):
    print('{}: {}'.format(letter, count))
```

另外，Counter实例还支持算数和集合操作。每次执行一个操作会生成一个新的Counter，并且计数为0或者为负数的元素会被删除。


# defaultdict
内建字典包含`setdefault`方法来获取默认值，而defaultdict会在初始化容器时就让调用者提前指定默认值。

```python
from collections import defaultdict
from functools import partial

d1 = defaultdict(int)
d2 = defaultdict(partial(int, 2))
d3 = defaultdict(partial(defaultdict, partial(int, 3)))
```


# OrderedDict
OrderedDict其实是字典的子类，它可以记住往字典中增加元素的顺序

需要注意的时，OrderedDict在检查相等性时，除了查看其内容，也会考虑元素增加的顺序


# deque
双端队列支持从任意一端增加和删除元素。栈和队列实际上就是双端队列的退化形式，其输入和输出都限制在一端。由于deque是一种序列容器，因此同样支持list的一些操作，如用`__getitem__`检查内容，确定长度，以及通过匹配标识从序列中删除某元素。

```python
from collections import deque

d = deque('abcdefg')
print(len(d))
print(d[0])
print(d[-1])

d.remove('a')
```

从左端或右端填充数据
```python
d1 = deque()
d1.extend('abc')
d1.append('efg')
d1.pop()

d2 = deque()
d2.extendleft('abc')
d2.appendleft('efg')
d2.popleft()
```

还可以按任意方向**旋转**从而跳过一些元素。
你可以形象地把deque中的元素想象成是刻在拨号盘上数字，当参数为负数，则逆时针(左旋)转动，否则为顺时针转动(右旋)。
```python
d.rotate(2)
d.rotate(-2)
```


# namedtuple
内建tuple使用数值索引来访问成员，一旦tuple中的字段变多，而且在代码实现中，可能元组的构造和使用相距很远，就很容易导致错误。namedtuple除了指定数值索引外，还回指定名字。在内存使用方面，namedtupel和内建元组同样高效。

```python
from collections import namedtuple

person = namedtuple('person', 'name age gender')

print('type of person: ', type(person))

rancho = person(name='rancho', age=20, gender='male')
print(rancho)
```

默认情况下，namedtuple定义字段时是不允许重复的，如果非要这么做，可以将`rename`参数设置为`True`

