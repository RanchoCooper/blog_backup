---
title: 时间和日期
catalog: true
date: 2017-10-03 13:20:16
subtitle: Dates and Times
header-img:
tags:
    - 札记
    - Python标准库
---

不同于int、float和str，Python没有包含对应日期和时间的内置类型，不过提供了3个相应模块，可以采用多种表示管理日期和时间值。

- `time`模块由底层C库提供与时间相关的函数。它包含一些函数用于获取时钟时间和处理器运行时间，还提供了基本的解析和字符串格式化工具。
- `datetime`模块为日期、时间以及日期时间提供了一个高层接口。datetime中的类支持算术、比较和时区配置。
- `calendar`模块可以创建周、月和年的格式化表示。它还可以计算重复事件、给定日期是周几，以及其他基于日历的值。

## 时钟时间 - time
`time`模块提供了一些用于管理日期和时间的C库函数。由于它绑定到底层C实现，一些细节(比如纪元开始时间和支持的最大日期)会特定于特定平台。


### 壁挂钟时间
`time`函数是time模块的核心函数之一，它会返回一个从纪元开始至今的浮点秒数。浮点数的表示对时间的存储和比较很有用，但是想要生成人类刻度的表示就差强人意了。如果你想记录或者打印时间，`ctime`可能更有用。

```python
import time
t = time.time()
print(t)
print(time.ctime())
print(time.ctime(t + 20))
```

### 处理器时间
`clock`的返回值反映了程序的实际使用CPU(运行)时间，应当用于性能测试、基准测试等。值得注意的是，如果程序什么也没做，这时候处理器时钟是不会走动的。

```python
import time

for i in range(10):
    print("%s %0.2f %0.2f" % (time.ctime(), time.time(), time.clock()))
    print("sleeping", i)
    time.sleep(i)
```
在这个例子里，每次循环仅仅进入睡眠，几乎没做什么工作。这样一来，两个挂壁时间在增加，而clock始终没发生变化。

### 时间组成
有些情况下需要将时间存储为过去了多少秒，还有的情况需要访问一个日期的各个字段(年、月等)。time模块定义了`struct_time`来维护日期和时间值。gmtime函数会返回UTC格式的当前时间，而localtime则会返回应用了当前时区的当前时间。mktime会取一个struct_time实例，并将其转换回浮点数表示。

```python
import time

def show_struct(t):
    print('tm_year: ', t.tm_year)
    print('tm_mon: ', t.tm_mon)
    print('tm_mday: ', t.tm_mday)       # day in month
    print('tm_hour: ', t.tm_hour)
    print('tm_min: ', t.tm_min)
    print('tm_sec: ', t.tm_sec)
    print('tm_wday: ', t.tm_wday)       # day in week
    print('tm_yday: ', t.tm_yday)       # day in year
    print('tm_isdst: ', t.tm_isdst)     # DST flag
    print('tm_zone: ', t.tm_zone)       # local time zone
    print('tm_gmtoff: ', t.tm_gmtoff)

show_struct(time.gmtime())
show_struct(time.localtime())
print('mktime: ', time.mktime(time.localtime()))
```

### 处理时区
确定当前时间的函数有一个前提，即已经设置了时区。时区的设置可能由应用程序设置，也可以使用系统的默认时区。修改时区不会改变具体的当前时间，只是改变了当前时间的表示(仅仅做了转换)。

要改变时区，需要设置环境变量TZ，然后调用tzset。设置时区时可以指定很多细节，甚至详细到日光的开始和结束时间。不过，通常的做法是使用时区名，并由底层库推导出其他信息。下面我们来简单地修改时区

```python
import os
import time

def show_zone_info():
    print('TZ: ', os.environ.get('TZ', '(not set yet)'))
    print('tzname: ', time.tzname)
    print('zone: %d (%d)' % (time.timezone, time.timezone / 3600))
    print('DST: ', time.daylight)
    print('Time: ', time.ctime())

print('Default:')
show_zone_info()

ZONES = ['GMT', 'Europe/Amsterdam']

for zone in ZONES:
    os.environ['TZ'] = zone
    time.tzset()
    print(zone, ': ')
    show_zone_info()
```

### 解析和格式化时间
工具函数strptime(parsed)和strftime(formatted)可以在struct_time和是兼职字符串表示之间转换


## 日期和时间值管理 - datetime

### 时间
时间值用`time`类表示，time实例包含hour、minute、second和microsecond属性，还可以包含时区信息。

```python
import datetime

t = datetime.time(1, 2, 3)

print(t)
print(t.hour)
print(t.minute)
print(t.second)
print(t.microsecond)
print(t.tzinfo)

print(datetime.time.min, datetime.time.max, datetime.time.resolution)
```

### 日期
日历日期值用`date`类表示，date实例包含year、month和day属性。使用`today`类方法很容易能穿件一个表示当前日期的日期实例。today类的`ctime`方法可以获得格式化的时间表示，而`timetuple`则返回today实例的struct_time结构。

```python
import datetime

today = datetime.date.today()
print(today)
print('ctime: ', today.ctime())
tt = today.timetuple()
print('tuple: tm_year = ', tt.tm_year)
```

另外，`datetime.date.fromordinal`和`datetime.date.fromtimestamp`分别可以将天数和时戳转换成对应的日期结构。

### timedelta
通过对两个datetime对象使用算数运算，或者结合使用`datetime`和`timedelta`，可以根据得到将来或过去的日期。比如，两个日期相减可以生成一个timedelta，对日期进行timedelta运算可以生成另一个日期。

实际上，通过`datetime.datetime.combine`方法，你可以将一个datetime.date实例和一个datetime.time相组合。

## 处理日期 - calendar
calendar模块实现了一些类来处理日期，管理面向年、月和周的值。通过prmonth方法可以生成一个月的格式化的文本日历输出。

```python
import calendar

c = calendar.TextCalendar(calendar.SUNDAY)      # 配置每周的起始日
c.prmonth(2017, 10)
```
