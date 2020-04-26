---
title: 漫谈Guava之RateLimiter
date: 2020-04-24 15:40:24
tags:
- Guava
- limiter
categories: Java


---

![](https://img.yjll.art/img/20200327141404.jpg)



​	日常开发中经常会出现一瞬间接口被超频次访问的这种情况，例如电商秒杀活动，定点抢红包等。短时间高并发可能会拖慢系统的响应速度，引起网络超时，一个服务不可用进而影响其他服务也不可用，甚至导致雪崩。

​	限流是最直接有效的控制手段，将系统处理不过来的请求拦截在核心逻辑之外。限流器实现方法有`漏桶算法(Leaky Bucket),`令牌桶算法(Token Bucket)`等。

<!-- more -->

* 漏斗算法 理解起来比较简单，水(即请求)流入固定大小的漏斗内，漏斗也已一定的速度流出水，当水满时停止流入,进行限流。
* 令牌桶算法 同样是固定大小的桶，定期往桶中投令牌，有新请求时需从桶中取令牌，当桶中无令牌可取时，则拒绝。

### 使用Guava  RateLimiter限流

​	本文使用`Guava`中的`RateLimiter`进行限流，我使用的Guava的版本为`28.2-jre`，不同版本api可能略有偏差。

```java

public class GuavaLimiterDemo {

    // 每秒生成2个令牌
    private static RateLimiter rateLimiter = RateLimiter.create(2);

    public static void main(String[] args)throws InterruptedException {
        ExecutorService executorService = Executors.newFixedThreadPool(100);
        // 5个线程同时调用，模拟多个请求同时访问
        IntStream.rangeClosed(1, 5).forEach(e -> {
            executorService.submit(() -> acquire(e));
        });
    }

    private static double acquire(int i){
        // acquire方法默认取一个令牌，桶中令牌不够的话会阻塞线程,返回值为线程阻塞时间
        double acquire = rateLimiter.acquire();
        System.out.printf("当前系统时间:%s,等待秒数:%s\n",LocalDateTime.now(),acquire);
        return acquire;
    }
}
```

​	调用静态方法`create()`创建限流器,参数为每秒生产令牌的数量

​	`acquire()`方法默认取一个令牌，可用调用重载的方法取多个令牌，如果桶中无令牌该方法会阻塞，开发过程中要注意大量线程挂起的问题，慎用。

​	`tryAcquire()`方法可解决上述问题，取一个令牌，如桶中无令牌直接返回false。
`tryAcquire(int permits, long timeout, TimeUnit unit)`可传入超时时间，超过这段时间桶中还是无令牌可取的话返回false。

```
当前系统时间:2020-03-25T13:30:37.757,等待秒数:0.0
当前系统时间:2020-03-25T13:30:38.195,等待秒数:0.458741
当前系统时间:2020-03-25T13:30:38.694,等待秒数:0.958729
当前系统时间:2020-03-25T13:30:39.194,等待秒数:1.458717
当前系统时间:2020-03-25T13:30:39.695,等待秒数:1.958704
```

​	这是控制台打印的结果，每秒放入两个令牌的桶，同时有5个线程访问。基本每0.5秒放行一个请求。

### Guava RateLimiter 实现原理



![](https://img.yjll.art/img/20200327141557.png)


​	RateLimiter限流器实现了两种算法`SmoothWarmingUp(漏斗算法)`和`SmoothBursty(令牌桶算法)`，默认使用`SmoothBursty`。

​	静态方法`RateLimiter.create()`默认使用子类`SmoothBursty`创建限流器

```java  
// permitsPerSecond 创建时传入的单位时间内发放令牌数量
public static RateLimiter create(double permitsPerSecond) {
    return create(permitsPerSecond, SleepingStopwatch.createFromSystemTimer());
  }
// stopwatch guava实现的时间监视器 默认SleepingStopwatch.createFromSystemTimer()
static RateLimiter create(double permitsPerSecond, SleepingStopwatch stopwatch) {
    // maxBurstSeconds 最大缓存时间，默认1.0
    RateLimiter rateLimiter = new SmoothBursty(stopwatch, 1.0 /* maxBurstSeconds */);
    // 初始化时设置投放令牌的速率
    rateLimiter.setRate(permitsPerSecond);
    return rateLimiter;
  }
```

​	`setRate`底层调用`doSetRate`设置速率

​	`setRate`为设置投放令牌的速率，调用时对以下值进行更改：

* storedPermits 当前令牌数量，速率更改时会根据新旧速率的比例对原桶中令牌的数量进行更改。初始化时为0

* stableIntervalMicros 每生产一个令牌需要的毫秒数

* maxPermits 桶中最多可存储的令牌数量，可根据maxBurstSeconds * permitsPerSecond 计算
  
* nextFreeTicketMicros 下一次可取令牌的时间，当前时间早于该时间表示可取元素，与之的差表示放入令牌且不取出令牌的时间，进而再跟据stableIntervalMicros可算出该时间段内桶中增加了多少令牌(受限于桶的大小)
  

​	在初始化时或手动更改速率时调用。

```java
// permitsPerSecond 创建时传入的单位时间内发放令牌数量
// nowMicros 从创建到现在经过的时间
final void doSetRate(double permitsPerSecond, long nowMicros) {
    resync(nowMicros);
    double stableIntervalMicros = SECONDS.toMicros(1L) / permitsPerSecond;
    this.stableIntervalMicros = stableIntervalMicros;
    doSetRate(permitsPerSecond, stableIntervalMicros);
  }
// stableIntervalMicros 每生产一个令牌，需要经过多少毫秒
void doSetRate(double permitsPerSecond, double stableIntervalMicros) {
      double oldMaxPermits = this.maxPermits;
      maxPermits = maxBurstSeconds * permitsPerSecond;
      if (oldMaxPermits == Double.POSITIVE_INFINITY) {
        // if we don't special-case this, we would get storedPermits == NaN, below
        storedPermits = maxPermits;
      } else {
        storedPermits =
            (oldMaxPermits == 0.0)
                ? 0.0 // initial state
                : storedPermits * maxPermits / oldMaxPermits;
      }
    }
```

​	速率更改或者从桶中取令牌时会调用`resync`方法

​	`resync(nowMicros)`将当前时间与`nextFreeTicketMicros`作比较，如当前时间在`nextFreeTicketMicros`之后，则重新计算桶中令牌的数量，并重置`nextFreeTicketMicros`的值。从这里可以看出Guava Limiter设计的绝妙之处，不需要单独开线程向桶中添加令牌，而是惰性的在取令牌时计算一下这时间内桶中添加了几个令牌，在根据桶中令牌数量进行限流。

```java
  void resync(long nowMicros) {
    // if nextFreeTicket is in the past, resync to now
    if (nowMicros > nextFreeTicketMicros) {
      double newPermits = (nowMicros - nextFreeTicketMicros) / coolDownIntervalMicros();
      // 防止添加后的令牌数量超过桶的极限
      storedPermits = min(maxPermits, storedPermits + newPermits);
      nextFreeTicketMicros = nowMicros;
    }
  }
```

​	取数据调用`acquire()`方法，桶中无令牌的话会阻塞

```java
  public double acquire(int permits) {
    // 需等待的时间
    long microsToWait = reserve(permits);
    // 线程挂起
    stopwatch.sleepMicrosUninterruptibly(microsToWait);
    return 1.0 * microsToWait / SECONDS.toMicros(1L);
  }
```

​	`acquire()`最终会调用`reserveEarliestAvailable()`

​	令牌桶限流器的一个优势是可以减缓突发情况带来的问题，允许单次请求的令牌数量超出剩余令牌数量，但是多拿的令牌数要从后边的补，正所谓“前人挖坑，后人填坑”。

```java
final long reserveEarliestAvailable(int requiredPermits, long nowMicros) {
    // 前文提到过，在取数据之前计算桶中令牌数量
    resync(nowMicros);
    long returnValue = nextFreeTicketMicros;
    double storedPermitsToSpend = min(requiredPermits, this.storedPermits);
    // 计算突发情况多取的令牌数量
    double freshPermits = requiredPermits - storedPermitsToSpend;
    long waitMicros =
        storedPermitsToWaitTime(this.storedPermits, storedPermitsToSpend)
            + (long) (freshPermits * stableIntervalMicros);
	// 超发的情况，下一次可取令牌的时间往后延
    this.nextFreeTicketMicros = LongMath.saturatedAdd(nextFreeTicketMicros, waitMicros);
    this.storedPermits -= storedPermitsToSpend;
    return returnValue;
  }
```



