---
title: CSharp容器源码之Queue
date: 2022-07-23 15:09:17
tags: [C#]
categories: 源码剖析
---
# 前言

复习下```Queue```的一些知识点。

[源码地址](https://referencesource.microsoft.com/#System/compmod/system/collections/generic/queue.cs)。

# 正文

### 定义
***
   队列, 先进先出(FIFO)的顺序容器,  底层为数组。

### 成员变量
***
修饰符|类型|变量名|作用简述|
--|:--:|--:|--:|
private|泛型数组| _array| 队列数组数据
private|int| _head| 队首下标位置    
private|int| _tail| 队尾下标位置     
private|int| _size| 队列元素数量 
private|int| _version| 迭代时用于数据版本校验
private|Object| _syncRoot| 多线程下的同步对象，已弃用，多线程建议使用ConcurrentQueue<T>
private static|泛型数组| _emptyArray|  默认构造函数初始化后指向的空数组

### 构造函数

TODO