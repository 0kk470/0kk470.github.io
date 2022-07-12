---
title: CSharp容器源码之Stack
date: 2022-07-12 23:19:57
tags: [C#]
categories: 源码剖析
---
# 前言

复习下```Stack```的一些知识点。

[源码地址](https://referencesource.microsoft.com/#System/compmod/system/collections/generic/stack.cs)。

# 正文

### 定义
***
   后进先出(LIFO)的顺序容器, 底层为数组。

### 成员变量
***
修饰符|类型|变量名|作用简述|
--|:--:|--:|--:|
private|T[]| _array|   栈数据 
private|int| _size|    栈内元素数量 
private|int| _version| 迭代时用于数据版本校验
private|Object| _syncRoot| 多线程下的同步对象，已弃用，多线程建议使用ConcurrentStack<T>
private static|T[]| _emptyArray = new T[0]|  默认构造函数初始化后指向的空数组

### 构造函数

共有三种构造函数，代码如下。

```CSharp
        public Stack() {
            _array = _emptyArray;
            _size = 0;
            _version = 0;
        }
    
        public Stack(int capacity) {
            if (capacity < 0)
                ThrowHelper.ThrowArgumentOutOfRangeException(ExceptionArgument.capacity, ExceptionResource.ArgumentOutOfRange_NeedNonNegNumRequired);
            _array = new T[capacity];
            _size = 0;
            _version = 0;
        }
    
        public Stack(IEnumerable<T> collection) 
        {
            if (collection==null)
                ThrowHelper.ThrowArgumentNullException(ExceptionArgument.collection);
 
            ICollection<T> c = collection as ICollection<T>;
            if( c != null) {
                int count = c.Count;
                _array = new T[count];
                c.CopyTo(_array, 0);  
                _size = count;
            }    
            else {                
                _size = 0;
                _array = new T[_defaultCapacity];                    
                
                using(IEnumerator<T> en = collection.GetEnumerator()) {
                    while(en.MoveNext()) {
                        Push(en.Current);                                    
                    }
                }
            }
        }
```

默认构造函数初始化会指向一个静态只读空数组，下一次```Push```一定会触发扩容。
传入Capacity的构造函数则会分配对应长度的数组。
传入```IEnumrable<T>```的构造函数会先尝试转换为```ICollection<T>```，如果成功了会调用```CopyTo```复制目标数组元素;否则的话就会分配默认大小(```defaultCapacity```为```4```)的数组容量

### 扩容机制

### 操作函数

# 总结