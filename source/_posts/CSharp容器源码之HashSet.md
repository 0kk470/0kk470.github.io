---
title: C#容器源码之HashSet
date: 2022-07-06 21:06:47
tags: [C#]
categories: 源码剖析
---
# 前言

复习下```HashSet```的一些知识点。

[源码地址](https://referencesource.microsoft.com/#System.Core/System/Collections/Generic/HashSet.cs)。

# 正文

### 定义
***
   不重复元素集合, 结构与```Dictionary```类似，底层为两个数组```buckets```和```slots```。

### 成员变量
***
修饰符|类型|变量名|作用简述|
--|:--:|--:|--:|
private|int数组| m_buckets| 位置映射数组
private|Slot数组|m_slots|存储所有要查找的元素的数组
private|int| m_count| 容器内当前的元素数量
private|int| m_lastIndex| 最新插入的集合元素的位置
private|int| m_version|数据版本，用于校验对数组的操作是否合法
private|int| m_freeList| 当前m_slots中指向的空节点
private|IEqualityComparer<T>| m_comparer| 用于查找时比较元素

### Slot结构
***
可以看到与```Dictionary```内部的```Entry```结构非常相似，只是少了一个为```Key```的成员变量。
```CSharp
        internal struct Slot {
            internal int hashCode;      // Lower 31 bits of hash code, -1 if unused
            internal int next;          // Index of next entry, -1 if last
            internal T value;
        }
```

### 构造函数
```CSharp
        public HashSet()
            : this(EqualityComparer<T>.Default) { }
 
        public HashSet(int capacity)
            : this(capacity, EqualityComparer<T>.Default) { }
 
        public HashSet(IEqualityComparer<T> comparer) {
            if (comparer == null) {
                comparer = EqualityComparer<T>.Default;
            }
 
            this.m_comparer = comparer;
            m_lastIndex = 0;
            m_count = 0;
            m_freeList = -1;
            m_version = 0;
        }
 
        public HashSet(IEnumerable<T> collection)
            : this(collection, EqualityComparer<T>.Default) { }
 
        /// <summary>
        /// Implementation Notes:
        /// Since resizes are relatively expensive (require rehashing), this attempts to minimize 
        /// the need to resize by setting the initial capacity based on size of collection. 
        /// </summary>
        /// <param name="collection"></param>
        /// <param name="comparer"></param>
        public HashSet(IEnumerable<T> collection, IEqualityComparer<T> comparer)
            : this(comparer) {
            if (collection == null) {
                throw new ArgumentNullException("collection");
            }
            Contract.EndContractBlock();
 
            var otherAsHashSet = collection as HashSet<T>;
            if (otherAsHashSet != null && AreEqualityComparersEqual(this, otherAsHashSet)) {
                CopyFrom(otherAsHashSet);
            }
            else {
                // to avoid excess resizes, first set size based on collection's count. Collection
                // may contain duplicates, so call TrimExcess if resulting hashset is larger than
                // threshold
                ICollection<T> coll = collection as ICollection<T>;
                int suggestedCapacity = coll == null ? 0 : coll.Count;
                Initialize(suggestedCapacity);
 
                this.UnionWith(collection);
 
                if (m_count > 0 && m_slots.Length / m_count > ShrinkThreshold) {
                    TrimExcess();
                }
            }
        }
 
        // Initializes the HashSet from another HashSet with the same element type and
        // equality comparer.
        private void CopyFrom(HashSet<T> source) {
            int count = source.m_count;
            if (count == 0) {
                // As well as short-circuiting on the rest of the work done,
                // this avoids errors from trying to access otherAsHashSet.m_buckets
                // or otherAsHashSet.m_slots when they aren't initialized.
                return;
            }
 
            int capacity = source.m_buckets.Length;
            int threshold = HashHelpers.ExpandPrime(count + 1);
            // 将被拷贝的hashset元素数量大的最小素数容量与当前hashset容量比较
            if (threshold >= capacity) {
                //当前容量更小就完全按照目标hashset大小容量拷贝
                m_buckets = (int[])source.m_buckets.Clone();
                m_slots = (Slot[])source.m_slots.Clone();
 
                m_lastIndex = source.m_lastIndex;
                m_freeList = source.m_freeList;
            }
            else {
                //当前容量更小就遍历目标hashset一个个添加
                int lastIndex = source.m_lastIndex;
                Slot[] slots = source.m_slots;
                Initialize(count);
                int index = 0;
                for (int i = 0; i < lastIndex; ++i)
                {
                    int hashCode = slots[i].hashCode;
                    if (hashCode >= 0)
                    {
                        AddValue(index, hashCode, slots[i].value);
                        ++index;
                    }
                }
                Debug.Assert(index == count);
                m_lastIndex = index;
            }
            m_count = count;
        }
 
        public HashSet(int capacity, IEqualityComparer<T> comparer)
            : this(comparer)
        {
            if (capacity < 0)
            {
                throw new ArgumentOutOfRangeException("capacity");
            }
            Contract.EndContractBlock();
 
            if (capacity > 0)
            {
                Initialize(capacity);
            }
        }
```

与```Dictionary```的构造函数逻辑类似， 关键函数还是分为以初始容量初始化和以另外的集合来初始化的逻辑。
但是在以其他集合初始化新的```HashSet```时，会分为两种情况:
1.以另一个具有相同元素类型和比较器的```HashSet```来初始化。这种情况下会直接调用```CopyFrom```从目标```HashSet```中拷贝所有元素。
```CSharp
        ICollection<T> coll = collection as ICollection<T>;
        int suggestedCapacity = coll == null ? 0 : coll.Count;
        Initialize(suggestedCapacity);
```

2.非```HashSet```的集合，如果实现```ICollection```接口，那么会先以```ICollection```的```Count```大小来初始化。然后调用```UnionWith```并集操作，
执行并集操作后如果HashSet已使用容量不到总容量的```1/3```(```ShrinkThreshold```常量为```3```)，则会调用裁剪函数优化```HashSet```底层存储数组的内存。
```CSharp
        if (m_count > 0 && m_slots.Length / m_count > ShrinkThreshold) {
            TrimExcess();
        }
```

### 扩容机制
***
在添加新元素时发现达到容量上限时触发，还是以大于```2倍当前容量```的最小素数为基准扩容，此处逻辑与[Dictionary的扩容逻辑](https://saltyfishkk.games/2022/06/30/CSharp%E5%AE%B9%E5%99%A8%E6%BA%90%E7%A0%81%E4%B9%8BDictionary/)非常相似，不赘述。代码如下

```CSharp
        private void IncreaseCapacity() {
            Debug.Assert(m_buckets != null, "IncreaseCapacity called on a set with no elements");
 
            int newSize = HashHelpers.ExpandPrime(m_count);
            if (newSize <= m_count) {
                throw new ArgumentException(SR.GetString(SR.Arg_HSCapacityOverflow));
            }
 
            // Able to increase capacity; copy elements to larger array and rehash
            SetCapacity(newSize, false);
        }
 
        /// <summary>
        /// Set the underlying buckets array to size newSize and rehash.  Note that newSize
        /// *must* be a prime.  It is very likely that you want to call IncreaseCapacity()
        /// instead of this method.
        /// </summary>
        private void SetCapacity(int newSize, bool forceNewHashCodes) { 
            Contract.Assert(HashHelpers.IsPrime(newSize), "New size is not prime!");
 
            Contract.Assert(m_buckets != null, "SetCapacity called on a set with no elements");
 
            Slot[] newSlots = new Slot[newSize];
            if (m_slots != null) {
                Array.Copy(m_slots, 0, newSlots, 0, m_lastIndex);
            }
 
            if(forceNewHashCodes) {
                for(int i = 0; i < m_lastIndex; i++) {
                    if(newSlots[i].hashCode != -1) {
                        newSlots[i].hashCode = InternalGetHashCode(newSlots[i].value);
                    }
                }
            }
 
            int[] newBuckets = new int[newSize];
            for (int i = 0; i < m_lastIndex; i++) {
                int bucket = newSlots[i].hashCode % newSize;
                newSlots[i].next = newBuckets[bucket] - 1;
                newBuckets[bucket] = i + 1;
            }
            m_slots = newSlots;
            m_buckets = newBuckets;
        }
```

### 关键函数
***

主要分为容器常用函数与集合操作函数
#### 容器常用函数

在```HashSet```容器中, ```Add```, ```Remove```, ```TryGetValue```, ```Contains```等函数的逻辑与前一篇文章中分析的[Dictionary的增删查逻辑](https://saltyfishkk.games/2022/06/30/CSharp%E5%AE%B9%E5%99%A8%E6%BA%90%E7%A0%81%E4%B9%8BDictionary/)非常类似，这里简单说明下即可。
概括下就是通过```bucket```位置找对应链表，通过比较链表元素中的```hashCode```的```Slot```(```Dictionary```里叫```Entry```)，添加的时候优先用缓存链表```freeList```的空```slot```，删除的时候将```slot```置回到```freeList```上。

代码就不再赘述, 接下来重点看下```HashSet```独有的一些操作函数。

#### 集合操作函数

此类函数都是基于离散数学中集合的概念来编写的。

##### 并集
将当前```HashSet```与另一个集合取并集，会修改当前```HashSet```, 时间复杂度```O(N)```

![union](union.png)

```CSharp
        public void UnionWith(IEnumerable<T> other) {
            if (other == null) {
                throw new ArgumentNullException("other");
            }
            Contract.EndContractBlock();
 
            foreach (T item in other) {
                AddIfNotPresent(item);
            }
        }
```

##### 交集
将当前```HashSet```与另一个集合取交集，会修改当前HashSet, 时间复杂度```O(N)```。

![intersect](intersect.png)

```CSharp
        public void IntersectWith(IEnumerable<T> other) {
            if (other == null) {
                throw new ArgumentNullException("other");
            }
            Contract.EndContractBlock();
 
            // intersection of anything with empty set is empty set, so return if count is 0
            if (m_count == 0) {
                return;
            }
 
            // if other is empty, intersection is empty set; remove all elements and we're done
            // can only figure this out if implements ICollection<T>. (IEnumerable<T> has no count)
            ICollection<T> otherAsCollection = other as ICollection<T>;
            if (otherAsCollection != null) {
                if (otherAsCollection.Count == 0) {
                    Clear();
                    return;
                }
 
                HashSet<T> otherAsSet = other as HashSet<T>;
                // faster if other is a hashset using same equality comparer; so check 
                // that other is a hashset using the same equality comparer.
                if (otherAsSet != null && AreEqualityComparersEqual(this, otherAsSet)) {
                    IntersectWithHashSetWithSameEC(otherAsSet);
                    return;
                }
            }
 
            IntersectWithEnumerable(other);
        }
```
这个函数有几个优化逻辑:
1. 如果目标集合是空```ICollection```的话，那么直接清空这个```HashSet```(与空集的交集就是空集本身)。
2. 如果目标集合是具有相同元素类型和比较器的```HashSet```，那么遍历移除掉目标集合已包含的元素，时间复杂度O(N)。
```CSharp
        /// <summary>
        /// If other is a hashset that uses same equality comparer, intersect is much faster 
        /// because we can use other's Contains
        /// </summary>
        /// <param name="other"></param>
        private void IntersectWithHashSetWithSameEC(HashSet<T> other) {
            for (int i = 0; i < m_lastIndex; i++) {
                if (m_slots[i].hashCode >= 0) {
                    T item = m_slots[i].value;
                    if (!other.Contains(item)) {
                        Remove(item);
                    }
                }
            }
        }
```
3. 如果目标集合仅仅是可迭代的，那么会先遍历目标集合，用一个```BitArray```来标记当前```HashSet```已包含的元素，然后再遍历当前```HashSet```移除掉所有标记的元素。
```CSharp

        /// <summary>
        /// Iterate over other. If contained in this, mark an element in bit array corresponding to
        /// its position in m_slots. If anything is unmarked (in bit array), remove it.
        /// 
        /// This attempts to allocate on the stack, if below StackAllocThreshold.
        /// </summary>
        /// <param name="other"></param>
        [System.Security.SecuritySafeCritical]
        private unsafe void IntersectWithEnumerable(IEnumerable<T> other) {
            Debug.Assert(m_buckets != null, "m_buckets shouldn't be null; callers should check first");
 
            // keep track of current last index; don't want to move past the end of our bit array
            // (could happen if another thread is modifying the collection)
            int originalLastIndex = m_lastIndex;
            int intArrayLength = BitHelper.ToIntArrayLength(originalLastIndex);
            BitHelper bitHelper;
            if (intArrayLength <= StackAllocThreshold) {
                int* bitArrayPtr = stackalloc int[intArrayLength];
                bitHelper = new BitHelper(bitArrayPtr, intArrayLength);
            }
            else {
                int[] bitArray = new int[intArrayLength];
                bitHelper = new BitHelper(bitArray, intArrayLength);
            }
 
            // mark if contains: find index of in slots array and mark corresponding element in bit array
            foreach (T item in other) {
                int index = InternalIndexOf(item);
                if (index >= 0) {
                    bitHelper.MarkBit(index);
                }
            }
 
            // if anything unmarked, remove it. Perf can be optimized here if BitHelper had a 
            // FindFirstUnmarked method.
            for (int i = 0; i < originalLastIndex; i++) {
                if (m_slots[i].hashCode >= 0 && !bitHelper.IsMarked(i)) {
                    Remove(m_slots[i].value);
                }
            }
        }

        /// <summary>
        /// Used internally by set operations which have to rely on bit array marking. This is like
        /// Contains but returns index in slots array. 
        /// </summary>
        /// <param name="item"></param>
        /// <returns></returns>
        private int InternalIndexOf(T item) {
            Debug.Assert(m_buckets != null, "m_buckets was null; callers should check first");
 
            int hashCode = InternalGetHashCode(item);
            for (int i = m_buckets[hashCode % m_buckets.Length] - 1; i >= 0; i = m_slots[i].next) {
                if ((m_slots[i].hashCode) == hashCode && m_comparer.Equals(m_slots[i].value, item)) {
                    return i;
                }
            }
            // wasn't found
            return -1;
        }
```

这个函数牺牲了部分空间(使用了```bitArray```来标记包含元素)来换取```O(N)```的时间复杂度，不然就得嵌套```for```循环以```O(N^2)```的时间复杂度来处理。
另外这个函数在```HashSet```元素数量小于等于```3170(StackAllocThreshold为100)```时会直接在栈上分配内存，超过后才会在堆上分配内存，
从而降低```GC```的开销。
```CSharp
            int intArrayLength = BitHelper.ToIntArrayLength(originalLastIndex);
            BitHelper bitHelper;
            if (intArrayLength <= StackAllocThreshold) {
                int* bitArrayPtr = stackalloc int[intArrayLength];
                bitHelper = new BitHelper(bitArrayPtr, intArrayLength);
            }

            internal static int ToIntArrayLength(int n) {
                return n > 0 ? ((n - 1) / IntSize + 1) : 0;
            }
```
该函数基于```BitArray位运算```来标记，一个```int```为```32位```，最多可以容纳```32```个byte标记, 实现如下:
```CSharp
    unsafe internal class BitHelper {   // should not be serialized
 
        private const byte MarkedBitFlag = 1;
        private const byte IntSize = 32;
 
        // m_length of underlying int array (not logical bit array)
        private int m_length;
        
        // ptr to stack alloc'd array of ints
        [System.Security.SecurityCritical]
        private int* m_arrayPtr;
 
        // array of ints
        private int[] m_array;
 
        // whether to operate on stack alloc'd or heap alloc'd array 
        private bool useStackAlloc;
 
        /// <summary>
        /// Instantiates a BitHelper with a heap alloc'd array of ints
        /// </summary>
        /// <param name="bitArray">int array to hold bits</param>
        /// <param name="length">length of int array</param>
        [System.Security.SecurityCritical]
        internal BitHelper(int* bitArrayPtr, int length) {
            this.m_arrayPtr = bitArrayPtr;
            this.m_length = length;
            useStackAlloc = true;
        }
 
        /// <summary>
        /// Instantiates a BitHelper with a heap alloc'd array of ints
        /// </summary>
        /// <param name="bitArray">int array to hold bits</param>
        /// <param name="length">length of int array</param>
        internal BitHelper(int[] bitArray, int length) {
            this.m_array = bitArray;
            this.m_length = length;
        }
 
        /// <summary>
        /// Mark bit at specified position
        /// </summary>
        /// <param name="bitPosition"></param>
        [System.Security.SecuritySafeCritical]
        internal unsafe void MarkBit(int bitPosition) {
            if (useStackAlloc) {
                int bitArrayIndex = bitPosition / IntSize;
                if (bitArrayIndex < m_length && bitArrayIndex >= 0) {
                    m_arrayPtr[bitArrayIndex] |= (MarkedBitFlag << (bitPosition % IntSize));
                }
            }
            else {
                int bitArrayIndex = bitPosition / IntSize;
                if (bitArrayIndex < m_length && bitArrayIndex >= 0) {
                    m_array[bitArrayIndex] |= (MarkedBitFlag << (bitPosition % IntSize));
                }
            }
        }
 
        /// <summary>
        /// Is bit at specified position marked?
        /// </summary>
        /// <param name="bitPosition"></param>
        /// <returns></returns>
        [System.Security.SecuritySafeCritical]
        internal unsafe bool IsMarked(int bitPosition) {
            if (useStackAlloc) {
                int bitArrayIndex = bitPosition / IntSize;
                if (bitArrayIndex < m_length && bitArrayIndex >= 0) {
                    return ((m_arrayPtr[bitArrayIndex] & (MarkedBitFlag << (bitPosition % IntSize))) != 0);
                }
                return false;
            }
            else {
                int bitArrayIndex = bitPosition / IntSize;
                if (bitArrayIndex < m_length && bitArrayIndex >= 0) {
                    return ((m_array[bitArrayIndex] & (MarkedBitFlag << (bitPosition % IntSize))) != 0);
                }
                return false;
            }
        }
 
        /// <summary>
        /// How many ints must be allocated to represent n bits. Returns (n+31)/32, but 
        /// avoids overflow
        /// </summary>
        /// <param name="n"></param>
        /// <returns></returns>
        internal static int ToIntArrayLength(int n) {
            return n > 0 ? ((n - 1) / IntSize + 1) : 0;
        }
    }
```
 ```BitArray```的结构如下图所示
 
 ![bitarray](bitarray.png)

 MarkBit就是将对应位置的int所对应的byte位置为1然后与原始值位或，IsMark就是将当前位置值取出来，进行位且运算判断目标byte位置是否为1。

 比如假设我们调用```MarkBit(137)```, 计算```137 / 32 = 4```，找到下标为```4```的位置的```int```数据， 然后计算 ```137 % 32 = 9```， 那么就将```1```左移到该int的第9位，然后位或运算将该```integer```的第9个二进制位置为1。

 如下图所示

  ![markbit](markbit.png)

同理 ```IsMark(137)```, 就是通过位且运算判断```index```为```4```位置的```integer```的```第9位二进制值```是否为```1```。
 

##### 差集

##### 对称差集

##### 子集

##### 真子集

##### 超集

##### 真超集

##### 集合重叠

### 总结
***
TODO