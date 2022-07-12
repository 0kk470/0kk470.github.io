---
title: C#容器源码之Stack
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
private|泛型数组| _array|   栈数据 
private|int| _size|    栈内元素数量 
private|int| _version| 迭代时用于数据版本校验
private|Object| _syncRoot| 多线程下的同步对象，已弃用，多线程建议使用ConcurrentStack<T>
private static|泛型数组| _emptyArray|  默认构造函数初始化后指向的空数组

### 构造函数

共有三种构造函数:

1. 默认构造函数初始化会指向一个静态只读空数组，下一次```Push```一定会触发扩容。
2. 传入Capacity的构造函数则会分配对应长度的数组。
3. 传入```IEnumrable<T>```的构造函数会先尝试转换为```ICollection<T>```，如果成功了会调用```CopyTo```复制目标数组元素; 否则的话就会分配默认大小(```defaultCapacity```为```4```)的数组容量，然后生成迭代器不停去```Push```, 期间可能会触发多次扩容。

代码如下
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

### 扩容机制

```Push```添加新元素前进行判断，容量满了会以当前大小的```2倍```来扩容

```CSharp
            if (_size == _array.Length) {
                T[] newArray = new T[(_array.Length == 0) ? _defaultCapacity : 2*_array.Length];
                Array.Copy(_array, 0, newArray, 0, _size);
                _array = newArray;
            }
```

### 操作函数

#### Push

向栈顶推入一个新元素，注意扩容触发的情况下时间复杂度为```O(N)```。

```CSharp
        public void Push(T item) {
            if (_size == _array.Length) {
                T[] newArray = new T[(_array.Length == 0) ? _defaultCapacity : 2*_array.Length];
                Array.Copy(_array, 0, newArray, 0, _size);
                _array = newArray;
            }
            _array[_size++] = item;
            _version++;
        }
```

#### Pop

#### Peek

#### Contains

#### Clear

#### TrimExcess

# 总结

1. 确定使用大小的情况下，使用预分配大小的构造函数，降低后续```Push```多次扩容的开销。

2. 尽量不用```IEnumrable<T>```来初始化```Stack```。

3. 在栈内数据未知的情况下，每次调用```Pop```或者```Peek```前先判断栈内是否有元素(```stack.Count > 0```)。