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

在```Push```添加新元素前进行判断，容量满了会以当前大小的```2倍```来扩容。

```CSharp
        if (_size == _array.Length) {
            T[] newArray = new T[(_array.Length == 0) ? _defaultCapacity : 2*_array.Length];
            Array.Copy(_array, 0, newArray, 0, _size);
            _array = newArray;
        }
```

为什么几乎所有的容器扩容的容量是上一次的```2倍```? 可以参考[知乎的讨论](https://www.zhihu.com/question/36538542)，以及```C++ vector```设计者[Andrew Koenig的解释](https://www.drdobbs.com/c-made-easier-how-vectors-grow/184401375)。

概括来说，对于n个元素的插入操作会导致K次扩容，那么```2```倍扩容拷贝元素的总次数为

![formula1](formula1.png)


即```O(N)```的时间复杂度, 扩容步骤如下图所示。

![capacity](capacity.png)

部分第三方实现会使用```1.5```倍作为扩容因子，这是为了```N```次扩容后，能够有机会复用之前的内存，对缓存更友好，不过在时间性能上会不如```2倍扩容```。而以```2倍```扩容，第N次需要的内存空间一定比前面```N - 1```次的扩容总量还要大，因此一定无法复用之前的内存空间。
两种因子的扩容比较参见下图。

![reuse](reuse.png)



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

弹出栈顶元素
```CSharp
        public T Pop() {
            if (_size == 0)
                ThrowHelper.ThrowInvalidOperationException(ExceptionResource.InvalidOperation_EmptyStack);
            _version++;
            T item = _array[--_size];
            _array[_size] = default(T);     // Free memory quicker.
            return item;
        }
```

#### Peek

获取栈顶元素

```CSharp
        public T Peek() {
            if (_size==0)
                ThrowHelper.ThrowInvalidOperationException(ExceptionResource.InvalidOperation_EmptyStack);
            return _array[_size-1];
        }
```

#### Contains

遍历栈检查是否包含指定元素，注意如果T是值类型的话，那么每次遍历会有装箱的开销，这种情况下应谨慎使用此函数。

```CSharp
        public bool Contains(T item) {
            int count = _size;
 
            EqualityComparer<T> c = EqualityComparer<T>.Default;
            while (count-- > 0) {
                if (((Object) item) == null) {
                    if (((Object) _array[count]) == null)
                        return true;
                }
                else if (_array[count] != null && c.Equals(_array[count], item) ) {
                    return true;
                }
            }
            return false;
        }
```

#### Clear

清空栈内所有元素

```CSharp
        public void Clear() {
            Array.Clear(_array, 0, _size);
            _size = 0;
            _version++;
        }
```

#### TrimExcess

将容量裁剪到当前元素数量大小，只会在元素数量超过容量```90%```的情况下调用才会生效。

```CSharp
        public void TrimExcess() {
            int threshold = (int)(((double)_array.Length) * 0.9);        
            if( _size < threshold ) {
                T[] newarray = new T[_size];
                Array.Copy(_array, 0, newarray, 0, _size);    
                _array = newarray;
                _version++;
            }
        }   
```

# 总结

1. 确定使用大小的情况下，尽可能地使用预分配大小的构造函数，以降低后续```Push```多次扩容的开销。

2. 尽量不用```IEnumrable<T>```来初始化```Stack```。

3. 在栈内数据未知的情况下，每次调用```Pop```或者```Peek```前先判断栈内是否有元素(```stack.Count > 0```)。