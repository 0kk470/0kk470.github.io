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
private|int|_version|数据版本，用于校验对数组的操作是否合法
private|int|_syncRoot_|多线程下使用的同步对象，不过现在基本废弃了，转用```ConcurrentBag```
private static| 泛型数组|_emptyArray| 默认构造函数内_items初始化指向的只读数组

## 构造函数

### 1. 默认构造函数
不会分配新的数组内存，而是指向一个默认的全局只读空数组，之后进行元素添加操作时才会触发扩容机制。
### 2. 带容量的构造函数
  会预分配参数```capacity```指定大小的数组。
### 3. 使用一组数据的构造函数
  如果传入的本身是一个容器```ICollection<T>```, 那么会使用这个容器的大小来初始化数组, 然后将容器数据元素拷贝到分配好的数组中去; 如果仅仅是```IEnumrable<T>```, 那么使用迭代器不停地将一个个元素```Add```，这其中会触发多次扩容。因此如果可以的话，尽量传入容器来初始化```List```。

## 扩容机制

   在向```List```中添加新元素时(调用```Add```、```Insert```、```AddRange```等)，会先调用一次```EnsureCapacity```来保证数组的容量足够容纳新的元素。如果是空数组，那么会以初始为```4```的容量来初始化，否则的话就会分配一个当前数组的长度```2```倍的新数组，然后再将当前数组的所有元素拷贝过去，时间复杂度为```O(n)```，因此在编写代码时，如果能预估列表使用大小的话，那么最好是提前预分配大小，减少频繁扩容带来的开销。

   ```CSharp
           private void EnsureCapacity(int min) {
            if (_items.Length < min) {
                int newCapacity = _items.Length == 0? _defaultCapacity : _items.Length * 2;
                if ((uint)newCapacity > Array.MaxArrayLength) newCapacity = Array.MaxArrayLength;
                if (newCapacity < min) newCapacity = min;
                Capacity = newCapacity;
            }
        }

         public int Capacity {
            get {
                Contract.Ensures(Contract.Result<int>() >= 0);
                return _items.Length;
            }
            set {
                if (value < _size) {
                    ThrowHelper.ThrowArgumentOutOfRangeException(ExceptionArgument.value, ExceptionResource.ArgumentOutOfRange_SmallCapacity);
                }
                Contract.EndContractBlock();
 
                if (value != _items.Length) {
                    if (value > 0) {
                        T[] newItems = new T[value];
                        if (_size > 0) {
                            Array.Copy(_items, 0, newItems, 0, _size);
                        }
                        _items = newItems;
                    }
                    else {
                        _items = _emptyArray;
                    }
                }
            }
        }
   ```

## 关键函数

### Insert
   先检测是否需要扩容，然后朝指定下标位置插入一个元素，会使用```Array.Copy```将目标位置之后的元素往后挪一位，因此不建议频繁在列表前段进行插入操作。
   ```Csharp
   public Insert(int index, Object item)
   {
      \*....*\
      if (index < _size) {
         Array.Copy(_items, index, _items, index + 1, _size - index);
      }
      \*....*\
   }
   ```
### InsertRange
   先检测扩容,然后尝试转换为```ICollection<T>```，如果不是```ICollection<T>```（即```IEnumrable<T>```)，那么会使用迭代器循环不停地去调用```Insert```（数量很大的话会有一定性能开销)。
   之后与```Insert```类似，将目标位置后元素往后挪，将插入的元素拷贝过去，拷贝时会生成一个临时数组来存放要拷贝的元素，这里有一个优化的地方时，如果被拷贝元素的容器和目标容器本来就是同一个，那么不需要分配新的临时数组，而是复用当前操作的数组。
   ```CSharp
      // If we're inserting a List into itself, we want to be able to deal with that.
   if (this == c) {
       // Copy first part of _items to insert location
       Array.Copy(_items, 0, _items, index, index);
       // Copy last part of _items back to inserted location
       Array.Copy(_items, index+count, _items, index*2, _size-index);
   }
   else {
       T[] itemsToInsert = new T[count];
       c.CopyTo(itemsToInsert, 0);
       itemsToInsert.CopyTo(_items, index);                    
   }
   ```
### Add
   先检测扩容，然后在当前列表末尾增添一个元素。还有一个非泛型的版本，传入的元素为```Object```，会尝试使用强制转换来添加新元素。
   ```Csharp
            try { 
                Add((T) item);            
            }
            catch (InvalidCastException) { 
                ThrowHelper.ThrowWrongValueTypeArgumentException(item, typeof(T));            
            }
   ```
### AddRange
   直接调用的```InsertRange```，传入的```index```为当前数组长度，即在数组末尾插入一组新的元素。
### RemoveAt
   将指定下标位置的元素移除，然后调用```Array.Copy```将后续元素往前挪
### Remove
   遍历```List```找到该元素的下标，如果存在则调用```RemoveAt``。
### RemoveRange
   从指定index位置开始移除count个元素，核心代码如下.
   ```Csharp
   if (count > 0) {
      int i = _size;
      _size -= count;
      if (index < _size) {
          Array.Copy(_items, index + count, _items, index, _size - index);
      }
      Array.Clear(_items, _size, count);
      _version++;
   }
   ```
### RemoveAll
这里算法有点意思，用了双指针的思路，先找到第一个需要被移除的元素下标，这里称为```freeindex```, 然后开始遍历循环这个```List```，每次循环都找到与之对应的不需要移除的第一个元素,这里称为```current```，然后将```current```元素移动到```freeindex```元素的位置, ```current```走到末尾循环结束，算法时间复杂度为```O(n)```, 之后就是将多余的元素Clear掉。
```Csharp
public int RemoveAll(Predicate<T> match) {
      if( match == null) {
          ThrowHelper.ThrowArgumentNullException(ExceptionArgument.match);
      }
      Contract.Ensures(Contract.Result<int>() >= 0);
      Contract.Ensures(Contract.Result<int>() <= Contract.OldValue(Count));
      Contract.EndContractBlock();

      int freeIndex = 0;   // the first free slot in items array

      // Find the first item which needs to be removed.
      while( freeIndex < _size && !match(_items[freeIndex])) freeIndex++;            
      if( freeIndex >= _size) return 0;
      
      int current = freeIndex + 1;
      while( current < _size) {
          // Find the first item which needs to be kept.
          while( current < _size && match(_items[current])) current++;            

          if( current < _size) {
              // copy item to the free slot.
              _items[freeIndex++] = _items[current++];
          }
      }                       
      
      Array.Clear(_items, freeIndex, _size - freeIndex);
      int result = _size - freeIndex;
      _size = freeIndex;
      _version++;
      return result;
  }
```
### IndexOf

找到目标元素在List中的第一次出现的下标位置, 有三个重载版本， 默认是整个数组(```Range[0, array.length]```)，如果传入```index```的话则是```Range[index, array.length]```，额外传入第三个参数```count```的话，则是从下标index开始后的count的元素(```Range[index, index + count]```)

### LastIndexOf

找到目标元素在List中的最后一次出现的下标位置，逻辑和参数与```IndexOf```类似

### Find

遍历```List```， 返回第一个满足条件函数的元素。

### FindIndex

与```Find```类似，返回第一个匹配的元素的下标, 可以向IndexOf一样传入额外两个参数```index```、```count```指定查找范围。

### FindLast

遍历```List```， 返回最后一个满足条件函数的元素，注意既然是最后一个元素，那么可以从数组末尾向首部查找。

### FindLastIndex

返回最后一个匹配的元素的下标

### Contains

遍历整个数组，判断某个元素是否在```List```中

### Exists

FindIndex的二次包装，判断```List```是否存在符合条件函数的元素。

### Sort

排序，具体的排序算法这里不赘述。

### Reverse

颠倒整个数组，两个重载版本，一个范围是整个数组，另一个版本的范围为```Range[index, index + count]```。

### TrueForAll

顾名思义，判断所有元素是否都符合条件函数。

### Clear

清空整个```List```

### TrimExcess

将数组容量(```Capacity```)裁剪到与当前元素数量(```Count```)一致，在数量达到容量90%的情况下不会生效。

```Csharp
        public void TrimExcess() {
            int threshold = (int)(((double)_items.Length) * 0.9);             
            if( _size < threshold ) {
                Capacity = _size;                
            }
        }
```  
## 使用总结

1. 尽量避免频繁在头部增删元素。
2. 不要在正向循环中去增删元素，否则会导致迭代的下标错位和越界。
3. 在```new```一个新的```Lis```t时，预估好容量，避免多次扩容的开销。另外使用一组数据元素初始化```List```时，避免使用```IEnumrable<T>```，尽可能地使用```ICollection<T>```。
4. 在确定```List```不会再扩容的情况下，可以使用TrimExcess裁剪一部分数组空位置

