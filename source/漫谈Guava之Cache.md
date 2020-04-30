---
title: 漫谈Guava之Cache（二）
date: 2020-03-28
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

LocalManualCache 相当于LocalCache的包装类，大部分方法直接使用LocalCache的实现，部分方法自己实现了一下。我们直接看LocalCache的逻辑，LocalCache的实现参考了`ConcurrentHashMap`，采用分段锁技术，当一个线程访问一个 `Segment` 时，不会影响其他的 `Segment`。

``` java
  Segment<K, V> segmentFor(int hash) {
    return segments[(hash >>> segmentShift) & segmentMask];
  }
``` 


Segment中维护了5个队列

* keyReferenceQueue 
* valueReferenceQueue 
* recencyQueue 
* writeQueue 
* accessQueue 

### 存放元素

加锁，判断加入的元素在缓存中是否存在

```java
V put(K key, int hash, V value, boolean onlyIfAbsent) {
  lock();
  try {
    long now = map.ticker.read();
    // 清理过期数据和被GC回收的数据
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

        // 表示触发GC，值被回收了
        if (entryValue == null) {
          ++modCount;
          // 排除CacheLoad加载的数据
          if (valueReference.isActive()) {
            // 移入删除队列中
            enqueueNotification(
                key, hash, entryValue, valueReference.getWeight(), RemovalCause.COLLECTED);
            setValue(e, key, value, now);
            newCount = this.count; // count remains unchanged
          } else {
            setValue(e, key, value, now);
            newCount = this.count + 1;
          }
          this.count = newCount; // write-volatile
          // 移除原来元素
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
    // 缓存中没有，创建一个
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
