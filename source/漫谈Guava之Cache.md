---
title: 漫谈Guava之Cache（二）
date: 2020-03-27 17:40:24
tags:
- Guava
- Cache
- 缓存
categories: Java
---


之前的文章介绍了Guava Cache的使用，本文将剖析Cache的源码，对Guava Cache的不太了解的可以移步这里：

[漫谈Guava之Cache（一）](https://blog.yjll.art/2020/03/27/%E6%BC%AB%E8%B0%88Guava%E4%B9%8BCache%EF%BC%88%E4%B8%80%EF%BC%89)



### 创建

创建Cache需通过CacheBuilder，该处使用了建造者模式，我们来看CacheBuilder的build方法。

```java
public <K1 extends K, V1 extends V> Cache<K1, V1> build() {
  checkWeightWithWeigher();
  checkNonLoadingCache();
  return new LocalCache.LocalManualCache<>(this);
}
```

```java
LocalManualCache(CacheBuilder<? super K, ? super V> builder) {
  this(new LocalCache<K, V>(builder, null));
}
```

LocalManualCache 相当于LocalCache的包装类，大部分方法直接使用LocalCache的实现，部分方法自己实现了一下。我们直接看LocalCache的逻辑


```java
LocalCache(
    CacheBuilder<? super K, ? super V> builder, @Nullable CacheLoader<? super K, V> loader) {
  concurrencyLevel = Math.min(builder.getConcurrencyLevel(), MAX_SEGMENTS);

  keyStrength = builder.getKeyStrength();
  valueStrength = builder.getValueStrength();

  keyEquivalence = builder.getKeyEquivalence();
  valueEquivalence = builder.getValueEquivalence();

  maxWeight = builder.getMaximumWeight();
  weigher = builder.getWeigher();
  expireAfterAccessNanos = builder.getExpireAfterAccessNanos();
  expireAfterWriteNanos = builder.getExpireAfterWriteNanos();
  refreshNanos = builder.getRefreshNanos();

  removalListener = builder.getRemovalListener();
  removalNotificationQueue =
      (removalListener == NullListener.INSTANCE)
          ? LocalCache.<RemovalNotification<K, V>>discardingQueue()
          : new ConcurrentLinkedQueue<RemovalNotification<K, V>>();

  ticker = builder.getTicker(recordsTime());
  entryFactory = EntryFactory.getFactory(keyStrength, usesAccessEntries(), usesWriteEntries());
  globalStatsCounter = builder.getStatsCounterSupplier().get();
  defaultLoader = loader;

  int initialCapacity = Math.min(builder.getInitialCapacity(), MAXIMUM_CAPACITY);
  if (evictsBySize() && !customWeigher()) {
    initialCapacity = (int) Math.min(initialCapacity, maxWeight);
  }

  // Find the lowest power-of-two segmentCount that exceeds concurrencyLevel, unless
  // maximumSize/Weight is specified in which case ensure that each segment gets at least 10
  // entries. The special casing for size-based eviction is only necessary because that eviction
  // happens per segment instead of globally, so too many segments compared to the maximum size
  // will result in random eviction behavior.
  int segmentShift = 0;
  int segmentCount = 1;
  while (segmentCount < concurrencyLevel && (!evictsBySize() || segmentCount * 20 <= maxWeight)) {
    ++segmentShift;
    segmentCount <<= 1;
  }
  this.segmentShift = 32 - segmentShift;
  segmentMask = segmentCount - 1;

  this.segments = newSegmentArray(segmentCount);

  int segmentCapacity = initialCapacity / segmentCount;
  if (segmentCapacity * segmentCount < initialCapacity) {
    ++segmentCapacity;
  }

  int segmentSize = 1;
  while (segmentSize < segmentCapacity) {
    segmentSize <<= 1;
  }

  if (evictsBySize()) {
    // Ensure sum of segment max weights = overall max weights
    long maxSegmentWeight = maxWeight / segmentCount + 1;
    long remainder = maxWeight % segmentCount;
    for (int i = 0; i < this.segments.length; ++i) {
      if (i == remainder) {
        maxSegmentWeight--;
      }
      this.segments[i] =
          createSegment(segmentSize, maxSegmentWeight, builder.getStatsCounterSupplier().get());
    }
  } else {
    for (int i = 0; i < this.segments.length; ++i) {
      this.segments[i] =
          createSegment(segmentSize, UNSET_INT, builder.getStatsCounterSupplier().get());
    }
  }
}
```


### 存放元素

`put`方法往cache中存放元素

```java
public V put(K key, V value) {
  checkNotNull(key);
  checkNotNull(value);
  int hash = hash(key);
  return segmentFor(hash).put(key, hash, value, false);
}
```

`segmentFor(hash)`根据hash及位运算，获取该key所在的Segment，这块的实现方式其实和 `ConcurrentHashMap `类似，采用分段锁技术，当一个线程访问一个 `Segment` 时，不会影响其他的 `Segment`。

其实真正的逻辑都是在`Segment`这个类的内部完成。

```java
V put(K key, int hash, V value, boolean onlyIfAbsent) {
  lock();
  try {
    long now = map.ticker.read();
    // 清理过期数据(非强引用)
    preWriteCleanup(now);
	
    int newCount = this.count + 1;
    // 扩容
    if (newCount > this.threshold) { // ensure capacity
      expand();
      newCount = this.count + 1;
    }

    AtomicReferenceArray<ReferenceEntry<K, V>> table = this.table;
    int index = hash & (table.length() - 1);
    ReferenceEntry<K, V> first = table.get(index);

    // Look for an existing entry.
    for (ReferenceEntry<K, V> e = first; e != null; e = e.getNext()) {
      K entryKey = e.getKey();
      if (e.getHash() == hash
          && entryKey != null
          && map.keyEquivalence.equivalent(key, entryKey)) {
        // We found an existing entry.

        ValueReference<K, V> valueReference = e.getValueReference();
        V entryValue = valueReference.get();

        if (entryValue == null) {
          ++modCount;
          if (valueReference.isActive()) {
            enqueueNotification(
                key, hash, entryValue, valueReference.getWeight(), RemovalCause.COLLECTED);
            setValue(e, key, value, now);
            newCount = this.count; // count remains unchanged
          } else {
            setValue(e, key, value, now);
            newCount = this.count + 1;
          }
          this.count = newCount; // write-volatile
          evictEntries(e);
          return null;
        } else if (onlyIfAbsent) {
          // Mimic
          // "if (!map.containsKey(key)) ...
          // else return map.get(key);
          recordLockedRead(e, now);
          return entryValue;
        } else {
          // clobber existing entry, count remains unchanged
          ++modCount;
          enqueueNotification(
              key, hash, entryValue, valueReference.getWeight(), RemovalCause.REPLACED);
          setValue(e, key, value, now);
          evictEntries(e);
          return entryValue;
        }
      }
    }

    // Create a new entry.
    // 当前key对应的ReferenceEntry不存在时创建一个，并把当前value和时间放入新建的ReferenceEntry中
    ++modCount;
    ReferenceEntry<K, V> newEntry = newEntry(key, hash, first);
    // 包装key，value放到newEntry中，并调用recordWrite()方法，更新权重,将entry存入accessQueue，writeQueue队列中
    setValue(newEntry, key, value, now);
    table.set(index, newEntry);
    newCount = this.count + 1;
    count用volatile修饰，保证内存可见性
    this.count = newCount; // write-volatile
    // TODO
    evictEntries(newEntry);
    return null;
  } finally {
    unlock();
    postWriteCleanup();
  }
}
```



```java
void recordWrite(ReferenceEntry<K, V> entry, int weight, long now) {
  // we are already under lock, so drain the recency queue immediately
  drainRecencyQueue();
  totalWeight += weight;

  if (map.recordsAccess()) {
    entry.setAccessTime(now);
  }
  if (map.recordsWrite()) {
    entry.setWriteTime(now);
  }
  accessQueue.add(entry);
  writeQueue.add(entry);
}
```

```
keyStrength

STRONG_ACCESS_WRITE

StrongAccessWriteEntry
```

com.google.common.cache.RemovalCause 删除原因
