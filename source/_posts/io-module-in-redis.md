---
title: Redis中的高性能IO模型
catalog: true
date: 2021-06-29 14:54:03
subtitle: IO Module In Redis
author: Rancho
header-img:
tags:
    - 存储中间件
    - Redis
---
本文主要探讨一个Redis中的经典问题: 为什么单线程的Redis能这么快?

首先, 要厘清一个事实, 我们通常所说的Redis是单线程, 主要是指Redis的网络IO和键值对读写都是由一个线程来完成的. 而Redis的其他功能, 比如持久化、异步删除、集群数据同步等, 其实是由额外的线程执行的. 所以, 严格来说Redis并不是单线程的

# Redis为什么用单线程

## 多线程的开销
日常写程序, 经常会听到一种说法: 使用多线程, 可以增加系统吞吐率, 或是可以增加系统扩展性. 的确, 对于一个多线程的系统来说, 在有合理的资源分配的情况下, 可以提升系统同时处理的请求数, 如下面的左图所示

但是, 通常情况下, 在外面采用多线程后, 如果没有良好的系统设计, 实际得到的结果, 其实是像右图所展示的那样. 刚开始增加线程数时, 系统吞吐会增加, 但是, 当超过一定的线程数时, 系统的吞吐就增长缓慢, 有时甚至还会出现下降的情况.

![](a.png)

之所以会出现这种情况, 关键的瓶颈在于当多个线程操作共享资源时, 为了保证一致性, 就需要额外的机制保证并发读写的正确性, 这样一来就会带来额外的开销. 并发访问控制是多线程开发中的一个难点问题, 如果没有精细的设计, 比如只是简单地采用一个粗粒度的互斥锁, 就会出现不理想的结果: 即使增加了线程数量, 大部分线程都在等待互斥锁, 并行变串行, 系统的吞吐并没有随之增加. 另外, 采用多线程开发一般会引入同步原语来保护共享资源的并发访问, 这也会降低系统代码的可维护性和易调试性. 为了避免这些问题, Redis直接采用了单线程模式

# 单线程Redis为什么那么快?
通常来说, 单线程的处理能力要比多线程差, 但Redis却能使用单线程模型达到每秒数十万级别的处理能力. 一方面因为Redis的大部分操作都在内存中完成, 再加上它采用了高效的数据结构, 例如哈希表和跳表. 另一方面, 就是Redis采用了`多路复用机制`, 使其在网络IO操作中能并发处理大量的客户端请求, 实现高吞吐率.

## 基本IO模型与阻塞点
在一次Redis命令的处理过程中, 涉及的网络操作包括监听请求(bind/listen), 建立连接(accept), 从socket中读取请求(recv), 最后返回结果(send). 在这些网络IO操作中, 潜在的阻塞点分别是accept()和recv()

![](b.png)

## 基于多路复用的IO模型
Linux中的IO多路复用是指一个线程处理多个IO流, 也就是select/epoll机制. 改机制允许内核同事存在多个监听套接字和已连接套接字, 内核会一直监听这些套接字上的连接请求, 一旦有请求到达就会交给Redis线程处理. 这样, Redis就不会阻塞在某个特定的客户端请求上

![](c.png)
select/epoll提供了基于事件的回调机制, 当select/epoll检测到FD上有请求到达时, 就会触发相应的事件. 这些事件会被放进一个事件队列, Redis线程不断处理该事件队列. 这样一来, Redis无需一直轮训是否有请求发生, 可以避免CPU资源浪费






