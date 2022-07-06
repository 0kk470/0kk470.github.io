---
title: CSharp容器源码之HashSet
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
 
            if (threshold >= capacity) {
                m_buckets = (int[])source.m_buckets.Clone();
                m_slots = (Slot[])source.m_slots.Clone();
 
                m_lastIndex = source.m_lastIndex;
                m_freeList = source.m_freeList;
            }
            else {
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
但是在以其他集合初始化新的```HashSet```时，会分为三种情况:
1.以另一个具有相同元素类型和比较器的```HashSet```来初始化。
2.以实现ICollection接口的集合来初始化。
3.未实现ICollection接口，比如IEnumrable这种来初始化。

### 扩容机制
***
TODO

### 集合的操作函数
***
TODO

### 总结
***
TODO