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

### 关键函数
***
主要就查找、添加、删除几个操作相关的函数，并辅以图像说明下。

假设先初始化了一个容量大小为5的哈希表
```CSharp
var dict = new Dictionary<string,int>(5);
```
那么它的初始结构如下图所示

![structure](1.png)

#### 插入
不管是```Add```也好还是直接通过操作符```[]```赋值,最终都会调用一个```Insert```函数，代码如下
```CSharp
        private void Insert(TKey key, TValue value, bool add) {
        
            if( key == null ) {
                ThrowHelper.ThrowArgumentNullException(ExceptionArgument.key);
            }
 
            if (buckets == null) Initialize(0);
            int hashCode = comparer.GetHashCode(key) & 0x7FFFFFFF;
            int targetBucket = hashCode % buckets.Length;
 
#if FEATURE_RANDOMIZED_STRING_HASHING
            int collisionCount = 0;
#endif
 
            for (int i = buckets[targetBucket]; i >= 0; i = entries[i].next) {
                if (entries[i].hashCode == hashCode && comparer.Equals(entries[i].key, key)) {
                    if (add) { 
                        ThrowHelper.ThrowArgumentException(ExceptionResource.Argument_AddingDuplicate);
                    }
                    entries[i].value = value;
                    version++;
                    return;
                } 
 
#if FEATURE_RANDOMIZED_STRING_HASHING
                collisionCount++;
#endif
            }
            int index;
            if (freeCount > 0) {
                index = freeList;
                freeList = entries[index].next;
                freeCount--;
            }
            else {
                if (count == entries.Length)
                {
                    Resize();
                    targetBucket = hashCode % buckets.Length;
                }
                index = count;
                count++;
            }
 
            entries[index].hashCode = hashCode;
            entries[index].next = buckets[targetBucket];
            entries[index].key = key;
            entries[index].value = value;
            buckets[targetBucket] = index;
            version++;
 
#if FEATURE_RANDOMIZED_STRING_HASHING
 
#if FEATURE_CORECLR
            // In case we hit the collision threshold we'll need to switch to the comparer which is using randomized string hashing
            // in this case will be EqualityComparer<string>.Default.
            // Note, randomized string hashing is turned on by default on coreclr so EqualityComparer<string>.Default will 
            // be using randomized string hashing
 
            if (collisionCount > HashHelpers.HashCollisionThreshold && comparer == NonRandomizedStringEqualityComparer.Default) 
            {
                comparer = (IEqualityComparer<TKey>) EqualityComparer<string>.Default;
                Resize(entries.Length, true);
            }
#else
            if(collisionCount > HashHelpers.HashCollisionThreshold && HashHelpers.IsWellKnownEqualityComparer(comparer)) 
            {
                comparer = (IEqualityComparer<TKey>) HashHelpers.GetRandomizedEqualityComparer(comparer);
                Resize(entries.Length, true);
            }
#endif // FEATURE_CORECLR
#endif
 
        }
```
可以看到如果之前并没有预分配容量(```buckets```为```null```)，那么会先通过```Intialize```函数给```buckets```和```entries```数组分配内存，之后就是嗲用```comapre.GetHashCode```分配哈希码，注意这里得到的哈希码要和最大正整数数```0x7FFFFFFF```进行位且运算来去掉符号位（不能是负数），这样之后通过和```buckets```的容量大小```length```进行模除运算就能达到插入的```哈希key```在```buckets```数组上映射的位置。之后就是分为两种情况:
1. 更新操作, 如果相同```bucket```位置的```entry```存在，并且```entry```的hashcode与计算后的hashcode一致，而且也不是```Add```操作，那么就更新```entry```存储的值，否则就会抛出重复添加元素的异常。
2. 插入操作, 会先检查有没有空的```entry```节点（```FreeCount``` > 0 ,``` Remove```操作后留下的空位置）, 如果有空节点那么会优先使用空节点```freeList```(```Remove```后留下的所有空节点会串连成一个缓存链表，这里的```freeList```指向的其实就是缓存链表的头节点)，然后使用头插法将其插入对应```bucket```位置指向的链表，否则就将count加1，然后选用entries[count]位置的节点

之后就是上文提到的判断哈希冲突是否过多从而进行重新哈希的逻辑。
举例来说，假如我们对上文的dict指向多次插入操作，而且它们的Hash码都是21，在bucket上映射的位置为1。
```CSharp
dict.Add("A", 1);
dict.Add("B", 2);
dict.Add("C", 3);
```
那么执行上述操作后，结构如下图

![2](2.png)

#### 删除
删除的逻辑比较简单, 先将```key```转换为```hashcode```，然后再转换为```buckets```中的位置，之后就是遍历比较```bucket```位置指向的链表比较```hashcode```，如果相等则清除对应位置的```entry```数据，并将其放入到```freelist```链表上缓存起来以方便下次```Insert```操作复用。代码如下:
```Csharp
        public bool Remove(TKey key) {
            if(key == null) {
                ThrowHelper.ThrowArgumentNullException(ExceptionArgument.key);
            }
 
            if (buckets != null) {
                int hashCode = comparer.GetHashCode(key) & 0x7FFFFFFF;
                int bucket = hashCode % buckets.Length;
                int last = -1;
                for (int i = buckets[bucket]; i >= 0; last = i, i = entries[i].next) {
                    if (entries[i].hashCode == hashCode && comparer.Equals(entries[i].key, key)) {
                        if (last < 0) {
                            buckets[bucket] = entries[i].next;
                        }
                        else {
                            entries[last].next = entries[i].next;
                        }
                        entries[i].hashCode = -1;
                        entries[i].next = freeList;
                        entries[i].key = default(TKey);
                        entries[i].value = default(TValue);
                        freeList = i;
                        freeCount++;
                        version++;
                        return true;
                    }
                }
            }
            return false;
        }
```

紧接着上文操作，假设我们执行下列删除操作。

```CSharp
dict.Remove("A");
dict.Remove("B");
```
那么结构变化如下图

删除```A```之后

![3](3.png)

删除```B```之后

![4](4.png)

如果之后又调用了
```CSharp
dict.Add("D", 4)
```
那么会将```freeList```指向的列表头节点复用起来，如果```D```的```HashCode```模除不为```1```，而是在其他位置，比如假设D的```HashCode```为```18```，即```bucket```位置为```3```。那么操作后的结构如下图

![5](5.png)


#### 查找

```GetOrDefault```、```Contains```、```TryGetValue```等函数内部都会用到```FindEntry```来查找对应元素。代码如下:

```Csharp
private int FindEntry(TKey key) {
            if( key == null) {
                ThrowHelper.ThrowArgumentNullException(ExceptionArgument.key);
            }
 
            if (buckets != null) {
                int hashCode = comparer.GetHashCode(key) & 0x7FFFFFFF;
                for (int i = buckets[hashCode % buckets.Length]; i >= 0; i = entries[i].next) {
                    if (entries[i].hashCode == hashCode && comparer.Equals(entries[i].key, key)) return i;
                }
            }
            return -1;
        }
```
核心逻辑就是通过对应的```bucket```位置找到链表，遍历比较```hashcode```即可。
所有操作大部分情况下时间复杂度皆为```O(1)```。

注意```ContainsValue```未使用映射关系查找，而是从头遍历，因此时间复杂度为```O(n)```。

```CSharp
        public bool ContainsValue(TValue value) {
            if (value == null) {
                for (int i = 0; i < count; i++) {
                    if (entries[i].hashCode >= 0 && entries[i].value == null) return true;
                }
            }
            else {
                EqualityComparer<TValue> c = EqualityComparer<TValue>.Default;
                for (int i = 0; i < count; i++) {
                    if (entries[i].hashCode >= 0 && c.Equals(entries[i].value, value)) return true;
                }
            }
            return false;
        }
```

### 使用总结
***
1. ```foreach```循环内不要去增删元素。
2. 如果要批量删除元素，用一个```List```存储好要删除的```key```，然后在```List```的循环中执行```Remove```操作。
3. 与```List```类似的扩容机制，所以在确定容量的情况下尽可能地预分配好空间。
4. 避免频繁增删元素，频繁的增删操作会导致```entries```的链表逻辑结构顺序变化，每次foreach的顺序也可能会因此不同。类似情况下建议换用```SortedDictionary```。