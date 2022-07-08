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
#### 常用操作函数
```Add```, ```Remove```, ```TryGetValue```, ```Contains```等函数的逻辑与前一篇文章中分析的[Dictionary的增删查逻辑](https://saltyfishkk.games/2022/06/30/CSharp%E5%AE%B9%E5%99%A8%E6%BA%90%E7%A0%81%E4%B9%8BDictionary/)非常类似，这里简单说明下即可。
概括下就是通过```bucket```位置找对应链表，通过比较链表元素中的```hashCode```的```Slot```(```Dictionary```里叫```Entry```)，添加的时候优先用缓存链表`freeList`的空```slot```，删除的时候将```slot```置回到```freeList```上。

代码就不再赘述, 接下来重点看下```HashSet```独有的一些操作函数。

#### 集合操作函数

此类函数都是基于离散数学中集合的概念来编写的。

### 并集
将当前HashSet与另一个集合取并集，会修改当前HashSet, 时间复杂度O(N)
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

### 交集
将当前HashSet与另一个集合取交集，会修改当前HashSet, 时间复杂度O(N)
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