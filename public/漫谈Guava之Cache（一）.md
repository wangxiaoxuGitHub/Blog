---
title: 漫谈Guava之Cache（一）
date: 2020-03-27 15:40:24
tags:
- Guava
- Cache
- 缓存
categories: Java
---

计算机的世界中，缓存无处不在，最常见的如CPU高速缓存，`CPU Cache`是用于减少[处理器](https://zh.wikipedia.org/wiki/中央处理器)访问内存所需平均时间的部件。在金字塔式[存储体系](https://zh.wikipedia.org/w/index.php?title=存储体系&action=edit&redlink=1)中它位于自顶向下的第二层，仅次于[CPU寄存器](https://zh.wikipedia.org/wiki/寄存器)。其容量远小于[内存](https://zh.wikipedia.org/wiki/内存)，但速度却可以接近处理器的频率。

当处理器发出内存访问请求时，会先查看缓存内是否有请求数据。如果存在（命中），则不经访问内存直接返回该数据；如果不存在（失效），则要先把内存中的相应数据载入缓存，再将其返回处理器。

这和我们日常开发中的很多场景相似，例如调用第三方接口取数据，微服务间取数据，甚至从数据库中取数据。这些IO操作所消耗的时间对于CPU来说是很久的。如果这些数据是读多写少，也就是说数据不会轻易改变，那我们可以把这部分数据进行缓存，取数据前先访问缓存，在缓存中查不到时再走原来的逻辑。


![cache](https://img.yjll.art/img/cache.jpg)

<!-- more -->

日常开发中我们可能会使用`ConcurrentHashMap`存数据，替代缓存使用，简单场景这样使用没什么问题，只是`ConcurrentHashMap`设计的目的不是做缓存，所以很多Cache应该有的很多功能，`ConcurrentHashMap`并不直接支持，类如设置key的过期时间，内存不足时释放缓存等。

本文介绍Guava的Cache使用，为该系列的第一部分。

我先写一个简单的例子，创建一个缓存，该缓存最多存100个元素，超时时间为10分钟

``` java
Cache<String, String> cache = CacheBuilder.newBuilder()
            .maximumSize(100)
            .expireAfterWrite(10, TimeUnit.MINUTES)
            .build();

cache.getIfPresent(key)

cache.get(key, () -> defaultValue)            

```

### CacheLoader
上文中的`cache.get(key, () -> defaultValue)`会在`cache`中没这条数据的情况下，根据我们写的策略重新计算，并将数据保存在缓存中一份。对于这种情况可以使用`CacheLoader`来统一处理。
```java
LoadingCache<Key, Graph> graphs = CacheBuilder.newBuilder()
      .expireAfterAccess(10, TimeUnit.MINUTES)
      .maximumSize(1000)
      .build(
          new CacheLoader<Key, Graph>() {
            public Graph load(Key key) { // no checked exception
              return createExpensiveGraph(key);
            }
          });
...
return graphs.getUnchecked(key);
```



## 三种淘汰策略

>Guava provides three basic types of eviction: size-based eviction, time-based eviction, and reference-based eviction.

### 容量大小

前文使用的`CacheBuilder.maximumSize(long)` 便可限制最大缓存数量。
如缓冲的每条数据有不同的权重的话可自己写`Weigher`策略，并设置最大`重量`。

```java
LoadingCache<Key, Graph> graphs = CacheBuilder.newBuilder()
       .maximumWeight(100000)
       .weigher(new Weigher<Key, Graph>() {
          public int weigh(Key k, Graph g) {
            return g.vertices().size();
          }
        })
       .build(
           new CacheLoader<Key, Graph>() {
             public Graph load(Key key) { // no checked exception
               return createExpensiveGraph(key);
             }
           });
```

### 时间

`CacheBuilder` provides two approaches to timed eviction:

*   `expireAfterAccess(long, TimeUnit)` 读或写多长时间后，缓存失效
*   `expireAfterWrite(long, TimeUnit)` 值更改后多长时间后，缓存失效


### 引用

使用JVM的垃圾回收，可被动的缓存失效。

* `CacheBuilder.weakKeys()` 对key使用弱引用包装
* `CacheBuilder.weakValues()` 对value使用弱引用包装
* `CacheBuilder.softValues()` 对soft使用软引用包装


## Removal Listeners
如果我们想对淘汰的元素进行处理，我们可以写一个`RemovalListener`对Cache进行监听。例如我们将DB连接存入cache中，当缓存失效时，我们需要关闭数据库连接，这部分逻辑就可以通过`RemovalListener`来完成

``` java
CacheLoader<Key, DatabaseConnection> loader = new CacheLoader<Key, DatabaseConnection> () {
  public DatabaseConnection load(Key key) throws Exception {
    return openConnection(key);
  }
};
RemovalListener<Key, DatabaseConnection> removalListener = new RemovalListener<Key, DatabaseConnection>() {
  public void onRemoval(RemovalNotification<Key, DatabaseConnection> removal) {
    DatabaseConnection conn = removal.getValue();
    conn.close(); // tear down properly
  }
};

CacheBuilder.newBuilder()
  .expireAfterWrite(2, TimeUnit.MINUTES)
  .removalListener(removalListener)
  .build(loader);
```

## Refresh

对于失效性比较强的元素，我们可以配置刷新的策略。刷新和淘汰不同，刷新元素是异步操作，刷新未完成时返回还是原数据，刷新成功则返回新数据。

``` java
// Some keys don't need refreshing, and we want refreshes to be done asynchronously.
LoadingCache<Key, Graph> graphs = CacheBuilder.newBuilder()
       .maximumSize(1000)
       .refreshAfterWrite(1, TimeUnit.MINUTES)
       .build(
           new CacheLoader<Key, Graph>() {
             public Graph load(Key key) { // no checked exception
               return getGraphFromDatabase(key);
             }

             public ListenableFuture<Graph> reload(final Key key, Graph prevGraph) {
               if (neverNeedsRefresh(key)) {
                 return Futures.immediateFuture(prevGraph);
               } else {
                 // asynchronous!
                 ListenableFutureTask<Graph> task = ListenableFutureTask.create(new Callable<Graph>() {
                   public Graph call() {
                     return getGraphFromDatabase(key);
                   }
                 });
                 executor.execute(task);
                 return task;
               }
             }
           });

```
