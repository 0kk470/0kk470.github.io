---
title: CSharp容器源码之SortedList
date: 2022-08-11 19:39:49
tags: [C#]
categories: 源码剖析
---
# 前言

复习下```SortedList```的一些知识点。

[源码地址](https://referencesource.microsoft.com/#System/compmod/system/collections/generic/sortedlist.cs)。

# 正文

## 定义
***
   按照键来排序的键/值对列表，可以通过键和索引来访问数据。连续性存储结构，底层为两个数组，可动态扩容。


## 成员变量
***
修饰符|类型|变量名|作用简述|
--|:--:|--:|--:|
private|TKey[]|keys| 键数据数组
private|TValue[]|values| 值数据数组
private|int|_size| 列表元素数量
private|int|_version|数据版本，用于校验对数组的操作是否合法
private|IComparer<TKey>|comparer| 排序用的比较函数
private |KeyList|keyList| 键列表，任何对该列表的操作也会影响到SortedList本身
private |ValueList|valueList| 值列表，任何对该列表的操作也会影响到SortedList本身
private|Object|_syncRoot_|多线程下使用的同步对象，不过现在基本废弃了。
private static| TKey[]|_emptyKeys| 默认构造函数内keys初始化指向的只读数组
private static| TValue[]|_emptyValues| 默认构造函数内values初始化指向的只读数组

## 构造函数

1. 默认构造函数。和其他容器的默认构造函数大同小异，初始数据数组默认指向空的只读数组，下次插入必定触发扩容。

```CSharp
        public SortedList() {
            keys = emptyKeys;
            values = emptyValues;
            _size = 0;
            comparer = Comparer<TKey>.Default; 
        }
```

2.  预分配容量的构造函数。

```CSharp
        public SortedList(int capacity) {
            if (capacity < 0)
                ThrowHelper.ThrowArgumentOutOfRangeException(ExceptionArgument.capacity, ExceptionResource.ArgumentOutOfRange_NeedNonNegNumRequired);
            keys = new TKey[capacity];
            values = new TValue[capacity];
            comparer = Comparer<TKey>.Default;
        }
```

3. 自定义比较器的构造，会先调用默认构造函数。

```CSharp
        public SortedList(IComparer<TKey> comparer) 
            : this() {
            if (comparer != null) {
                this.comparer = comparer;
            }                
        }
```

4. 自定义比较器以及分配容量的构造函数，会触发一次扩容。

```CSharp
        public SortedList(int capacity, IComparer<TKey> comparer) 
            : this(comparer) {
            Capacity = capacity;
        }
```

5. 用另一个字典以及自定义比较器来初始化的构造函数, 调用了第4种构造函数，然后将数据拷贝过去， 并进行一次排序。

```CSharp
        public SortedList(IDictionary<TKey, TValue> dictionary, IComparer<TKey> comparer) 
            : this((dictionary != null ? dictionary.Count : 0), comparer) {
            if (dictionary==null)
                ThrowHelper.ThrowArgumentNullException(ExceptionArgument.dictionary);
 
            dictionary.Keys.CopyTo(keys, 0);
            dictionary.Values.CopyTo(values, 0);
            Array.Sort<TKey, TValue>(keys, values, comparer);
            _size = dictionary.Count;            
        }
```

6. 仅使用另一个字典初始化，调用的第5个构造函数。

```CSharp
        public SortedList(IDictionary<TKey, TValue> dictionary) 
            : this(dictionary, null) {
        }
```

## 扩容机制

空数组情况下第一次扩容为```4```，其余情况为```2```倍扩容

```CSharp
        private void EnsureCapacity(int min) {
            int newCapacity = keys.Length == 0? _defaultCapacity: keys.Length * 2;
            // Allow the list to grow to maximum possible capacity (~2G elements) before encountering overflow.
            // Note that this check works even when _items.Length overflowed thanks to the (uint) cast
            if ((uint)newCapacity > MaxArrayLength) newCapacity = MaxArrayLength;
            if (newCapacity < min) newCapacity = min;
            Capacity = newCapacity;
        }
```

## 关键函数

### 查找

一般是根据```key```来找值，当然也支持通过数组下标来找值，使用```ValueList[idx]```访问即可。

```CSharp
        public int IndexOf(TKey key) {
            if (((Object) key) == null)
                ThrowHelper.ThrowArgumentNullException(ExceptionArgument.key);
    
            int i = Array.BinarySearch<TKey>(_dict.keys, 0,
                                      _dict.Count, key, _dict.comparer);
            if (i >= 0) return i;
            return -1;
        }

        public bool ContainsKey(TKey key) {
            return IndexOfKey(key) >= 0;
        }
    
        public bool ContainsValue(TValue value) {
            return IndexOfValue(value) >= 0;
        }
```
因为是列表本身是有序的，所以```key```用二分查找会更快，时间复杂度```O(logN)```。

但是```value```的查找还是得用数组遍历来处理，时间复杂度```O(N)```。

另外也可以根据下标来找```key```。

```CSharp
        private TKey GetKey(int index) {
            if (index < 0 || index >= _size) 
               ThrowHelper.ThrowArgumentOutOfRangeException(ExceptionArgument.index, ExceptionResource.ArgumentOutOfRange_Index);
            return keys[index];
        }
```

### 插入

可以像```Dictionary```那样来赋值来插入数据。

```CSharp
        public TValue this[TKey key] {
            get {
                int i = IndexOfKey(key);
                if (i >= 0)
                    return values[i];
 
                ThrowHelper.ThrowKeyNotFoundException();
                return default(TValue);
            }
            set {
                if (((Object) key) == null) ThrowHelper.ThrowArgumentNullException(ExceptionArgument.key);
                int i = Array.BinarySearch<TKey>(keys, 0, _size, key, comparer);
                if (i >= 0) {
                    values[i] = value;
                    version++;
                    return;
                }
                Insert(~i, key, value);
            }
        }

        private void Insert(int index, TKey key, TValue value) {
            if (_size == keys.Length) EnsureCapacity(_size + 1);
            if (index < _size) {
                Array.Copy(keys, index, keys, index + 1, _size - index);
                Array.Copy(values, index, values, index + 1, _size - index);
            }
            keys[index] = key;
            values[index] = value;
            _size++;
            version++;
        }        
```

如果二分查找找不到```key```，则会返回合适得插入位置下标的取反，取反返回的必是负数（代表列表中不存在这个```key```）。

插入位置的选取参见官方文档说明
![1](1.png)

大意就是如果存在比要插入的```key```更大的```key_larger```，则返回第一个```key_larger```的下标取反值。如果没有更大的键，则返回列表最后一个元素下标的取反值。

之后就是将目标位置之后的元素后挪```1```位，腾个地给新来的```key```和```value```。

### 删除

既可以根据下标删除，也可以根据```key```来删除。

```CSharp
        public void RemoveAt(int index) {
            if (index < 0 || index >= _size) ThrowHelper.ThrowArgumentOutOfRangeException(ExceptionArgument.index, ExceptionResource.ArgumentOutOfRange_Index);
            _size--;
            if (index < _size) {
                Array.Copy(keys, index + 1, keys, index, _size - index);
                Array.Copy(values, index + 1, values, index, _size - index);
            }
            keys[_size] = default(TKey);
            values[_size] = default(TValue);
            version++;
        }
    

        public bool Remove(TKey key) {
            int i = IndexOfKey(key);
            if (i >= 0) 
                RemoveAt(i);
            return i >= 0;
        }
```
注意```key```不能为空。

### 裁剪容量

与```List```一致，容量使用未达到```90%```就可以裁剪。

```CSharp
        public void TrimExcess() {
            int threshold = (int)(((double)keys.Length) * 0.9);             
            if( _size < threshold ) {
                Capacity = _size;                
            }
        }
```

## 总结

1. 如果要使用数组下标访问数据可以通过```ValueList```和```KeyList```两个索引器来访问，但需要注意的是这两个接口虽然是继承自```IList<T>```，但只能读数据，任何写操作(```Remove,Insert,Clear,Add```)都不被允许。

2. 与其他容器一样，能预估容量提前分配就使用预分配构造函数。

3. 在少量键值对数据需要保持有序的情况下使用该容器。