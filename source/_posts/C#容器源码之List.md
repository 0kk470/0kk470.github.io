---
title: C#容器源码之List
date: 2022-06-09 22:33:49
tags: [C#]
categories: 源码剖析
---
# 前言

复习下```List```的一些知识点。

[源码地址](https://referencesource.microsoft.com/#mscorlib/system/collections/generic/list.cs)。

# 正文

## 定义

   可通过索引访问的对象的强类型列表，连续性存储结构，底层为数组，可动态扩容。


## 成员变量

修饰符|类型|变量名|作用简述|
--|:--:|--:|--:|
private|泛型数组|_items_| 存储目标数据类型的数组
private|int|_size_|当前数组内的条目数量
private|int|_version|数据版本，用于保证对数组的合法操作
private|int|_syncRoot_|多线程下使用的同步对象，不过现在基本废弃了，转用```ConcurrentBag```
private static| 泛型数组|_emptyArray| 默认构造函数内_items初始化指向的只读数组

## 构造函数

TODO

## 核心函数

TODO

## 使用总结

TODO

