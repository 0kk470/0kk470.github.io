---
title: C#容器源码之Dictionary
date: 2022-06-30 13:15:31
tags: [C#]
categories: 源码剖析
---
# 前言

复习下```Dictionary```的一些知识点。

[源码地址](https://referencesource.microsoft.com/#mscorlib/system/collections/generic/dictionary.cs)。

# 正文

### 定义
***
   泛型的哈希表, 通过把关键码值映射到表中一个位置来访问数据，以加快查找的速度，是一种键值对关系的集合，底层为两个数组```buckets```和```entries```。

### 成员变量
***
修饰符|类型|变量名|作用简述|
--|:--:|--:|--:|
private|int数组| buckets| hashkey映射数组
private|Entry数组|entries|存储所有要查找的值的数组
private|int| count| 容器内当前的元素数量
private|int| version|数据版本，用于校验对数组的操作是否合法
private|int| freeList| 当前entries中指向的空节点
private|int| freeCount| 空节点数量
private|IEqualityComparer<TKey>| comparer| 用于查找时比较hashkey
private|KeyCollection| keys| 键集合，惰性生成
private|ValueCollection| values| 值集合，惰性生成
private|Object| _syncRoot|多线程下使用的同步对象，不过现在基本废弃了， 多线程建议使用```ConcurrentDictionary```

### Entry结构

```CSharp
        private struct Entry {
            public int hashCode;    // Lower 31 bits of hash code, -1 if unused
            public int next;        // Index of next entry, -1 if last
            public TKey key;           // Key of entry
            public TValue value;         // Value of entry
        }
```

虽然数据都是存储在一个名为```entries```的数组中，但是```entry```之间的逻辑结构实际上为链表结构, 即相同```bucket```映射位置的```entry```会通过```next```"指针"串联起来。至于为什么不直接用链表，个人猜测是使用数组能够保证一段内存的连续性，而真的使用纯指针链表来访问，内存数据会变得更碎片化，更容易cache miss，从而导致访问效率变低。

### 构造函数
***
```CSharp
        public Dictionary(): this(0, null) {}
 
        public Dictionary(int capacity): this(capacity, null) {}
 
        public Dictionary(IEqualityComparer<TKey> comparer): this(0, comparer) {}
 
        public Dictionary(int capacity, IEqualityComparer<TKey> comparer) {
            if (capacity < 0) ThrowHelper.ThrowArgumentOutOfRangeException(ExceptionArgument.capacity);
            if (capacity > 0) Initialize(capacity);
            this.comparer = comparer ?? EqualityComparer<TKey>.Default;
 
#if FEATURE_CORECLR
            if (HashHelpers.s_UseRandomizedStringHashing && comparer == EqualityComparer<string>.Default)
            {
                this.comparer = (IEqualityComparer<TKey>) NonRandomizedStringEqualityComparer.Default;
            }
#endif // FEATURE_CORECLR
        }
 
        public Dictionary(IDictionary<TKey,TValue> dictionary): this(dictionary, null) {}
 
        public Dictionary(IDictionary<TKey,TValue> dictionary, IEqualityComparer<TKey> comparer):
            this(dictionary != null? dictionary.Count: 0, comparer) {
 
            if( dictionary == null) {
                ThrowHelper.ThrowArgumentNullException(ExceptionArgument.dictionary);
            }
 
            foreach (KeyValuePair<TKey,TValue> pair in dictionary) {
                Add(pair.Key, pair.Value);
            }
        }
```

可以看到关键的构造函数主要为以下两个函数，一个预分配容量，一个以另一个键值对容器初始化。
```CSharp
public Dictionary(int capacity, IEqualityComparer<TKey> comparer)

public Dictionary(IDictionary<TKey,TValue> dictionary,IEqualityComparer<TKey> comparer)

```
预分配构造函数将```bucket```和```entries```两个数组以```capacity```大小初始化，而以另一容器作为参数的构造函数则不停循环将参数容器的元素依次```Add```到新创建的```Dictionary```当中。

```comparer```为hashkey的查找比较函数，传入```null```即使用默认比较函数```EqualityComparer<TKey>.Default```。

### 扩容机制
***
扩容的时机与```List```类似，即在``Insert``插入新元素时如果容器元素数量达到上限触发，关键代码如下。

```CSharp
        private void Resize() {
            Resize(HashHelpers.ExpandPrime(count), false);
        }
 
        private void Resize(int newSize, bool forceNewHashCodes) {
            Contract.Assert(newSize >= entries.Length);
            int[] newBuckets = new int[newSize];
            for (int i = 0; i < newBuckets.Length; i++) newBuckets[i] = -1;
            Entry[] newEntries = new Entry[newSize];
            Array.Copy(entries, 0, newEntries, 0, count);
            if(forceNewHashCodes) {
                for (int i = 0; i < count; i++) {
                    if(newEntries[i].hashCode != -1) {
                        newEntries[i].hashCode = (comparer.GetHashCode(newEntries[i].key) & 0x7FFFFFFF);
                    }
                }
            }
            for (int i = 0; i < count; i++) {
                if (newEntries[i].hashCode >= 0) {
                    int bucket = newEntries[i].hashCode % newSize;
                    newEntries[i].next = newBuckets[bucket];
                    newBuckets[bucket] = i;
                }
            }
            buckets = newBuckets;
            entries = newEntries;
        }
```
其中扩容调用的是```Resize()```函数，在通过```ExpandPrime```函数确定新容量大小后，则会分配重新分配buckets和entries的数组大小，并将数据Copy过去，另外entry之间

在这有一个扩容的优化算法来降低哈希冲突: 分配的新容器大小为与 ```2倍于目前容器容量大小``` 最接近的``最小素数``。
```CSharp
        public static readonly int[] primes = {
            3, 7, 11, 17, 23, 29, 37, 47, 59, 71, 89, 107, 131, 163, 197, 239, 293, 353, 431, 521, 631, 761, 919,
            1103, 1327, 1597, 1931, 2333, 2801, 3371, 4049, 4861, 5839, 7013, 8419, 10103, 12143, 14591,
            17519, 21023, 25229, 30293, 36353, 43627, 52361, 62851, 75431, 90523, 108631, 130363, 156437,
            187751, 225307, 270371, 324449, 389357, 467237, 560689, 672827, 807403, 968897, 1162687, 1395263,
            1674319, 2009191, 2411033, 2893249, 3471899, 4166287, 4999559, 5999471, 7199369};

        /*......Other Code......*/

        // Returns size of hashtable to grow to.
        public static int ExpandPrime(int oldSize)
        {
            int newSize = 2 * oldSize;
 
            // Allow the hashtables to grow to maximum possible size (~2G elements) before encoutering capacity overflow.
            // Note that this check works even when _items.Length overflowed thanks to the (uint) cast
            if ((uint)newSize > MaxPrimeArrayLength && MaxPrimeArrayLength > oldSize)
            {
                Contract.Assert( MaxPrimeArrayLength == GetPrime(MaxPrimeArrayLength), "Invalid MaxPrimeArrayLength");
                return MaxPrimeArrayLength;
            }
 
            return GetPrime(newSize);
        }

        public static int GetPrime(int min) 
        {
            if (min < 0)
                throw new ArgumentException(Environment.GetResourceString("Arg_HTCapacityOverflow"));
            Contract.EndContractBlock();
 
            for (int i = 0; i < primes.Length; i++) 
            {
                int prime = primes[i];
                if (prime >= min) return prime;
            }
 
            //outside of our predefined table. 
            //compute the hard way. 
            for (int i = (min | 1); i < Int32.MaxValue;i+=2) 
            {
                if (IsPrime(i) && ((i - 1) % Hashtable.HashPrime != 0))
                    return i;
            }
            return min;
        }
```
为什么每次扩容的容量大小都为素数?

关于这个问题可以参考[stackoverflow上的讨论](https://stackoverflow.com/questions/4638520/why-net-dictionaries-resize-to-prime-numbers)以及[这篇博客](https://blog.csdn.net/zhishengqianjun/article/details/79087525#23)。

结论就是: 在大多数情况下，对于一组数列中的任意数字```n```，如果它的公因数数量越少，那么该数列经散列后的新数字在新的哈希数组上分布就越均匀，即发生哈希冲突的概率就越小。而素数的公因数为```1```和它本身，公因数最少，所以每次扩容都尽可能地选择素数作为新容量大小。

### 哈希冲突过多的优化

在```Insert```时发生了过多哈希冲突（超过```100次```）则会调用```Resize```来重新获取所有``hashcode``。

```Csharp
            if(collisionCount > HashHelpers.HashCollisionThreshold && HashHelpers.IsWellKnownEqualityComparer(comparer)) 
            {
                comparer = (IEqualityComparer<TKey>) HashHelpers.GetRandomizedEqualityComparer(comparer);
                Resize(entries.Length, true);
            }
```

关键函数

TODO