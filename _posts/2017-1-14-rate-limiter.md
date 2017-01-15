---
layout: post
categories: [架构]
tags: [架构]
code: true
title: 限流方案的设计
---

        有些场景，比如秒杀、写服务和特别频繁复杂的查询，系统扛不住，需要一种手段控制并发/请求量，就是限流，限流也叫流量整形（Traffic Shaping），是一种主动调整流量输出速率的措施，当请求过多的时候可以采取直接拒绝服务或者阻塞的形式来对付。在2016年10月份看了很多资料，写了一个小项目在这个github上，打算重新整理成笔记好日后查阅，

### 算法介绍
#### 1. 计数器法
​        计数器法是指自然单位时间内统计次数，时间进入下一个单位时间则重置大小，重新计数。适用于不需要那么严格限流的场景。这个方法最简单有效，是项目中用的最多的了，比方一分钟内输入密码错误三次则要求输入验证码。

这里介绍一下Redis [*INCR*](http://doc.redisfans.com/string/incr.html#incr) 和Dubbo [TPSLimitFilter](https://github.com/alibaba/dubbo/blob/27917f2e86bbd97ee047d69817730a57bdf5ad6b/dubbo-rpc/dubbo-rpc-api/src/main/java/com/alibaba/dubbo/rpc/filter/tps/StatItem.java)

Redis官方文档INCR命令：在这里也简单引用一下：典型用法是限制公开 API 的请求次数，以下是一个限速器实现示例，它将 API 的最大请求数限制在每个 IP 地址每秒钟十个之内：

```  c
FUNCTION LIMIT_API_CALL(ip)
ts = CURRENT_UNIX_TIME()
keyname = ip+":"+ts
current = GET(keyname)
IF current != NULL AND current > 10 THEN
    ERROR "too many requests per second"
ELSE
    MULTI
        INCR(keyname,1)
        EXPIRE(keyname,10)
    EXEC
    PERFORM_API_CALL()
END
   
```
​        我看Redis中文文档中将原文“EXPIRE(keyname,10)”写成了“EXPIRE(keyname,1)”，我认为这两种写法在特殊场景下会造成限流结果的不同，因为多个JVM的机器的时钟可能会有细微差别（因为各个机器时钟的差别这种方式不能真正意义上的精确地控制每一秒的请求数量，因为时间戳是JVM所在机器提供的，不是Redis提供，而且发送的时候还有网络延时。当然，计数器法本就是用于不严格限流的场景），而且上述代码时间戳的获得和GET命令的时间可能不是同一秒，可能GET了上一秒的结果，而上一秒如果是跑的最快的那个机器在那秒的开始了执行了“EXPIRE(keyname,1)”，这时候最慢的那台机器去获得的是这个key过期了，再次执行事务会该key会从1开始重新计算，而如果这一秒超过了最大请求次数，显然按照英文文档的做法是不允许通过，而中文文档该key刚好过期了是可以通过的。这里用了事务打包来避免引入竞争条件，还有不用事务的方式：

```c
FUNCTION LIMIT_API_CALL(ip):
current = GET(ip)
IF current != NULL AND current > 10 THEN
    ERROR "too many requests per second"
ELSE
    value = INCR(ip)
    IF value == 1 THEN
        EXPIRE(ip,1)
    END
    PERFORM_API_CALL()
END
```

​        而作者认为在 [*INCR*](http://doc.redisfans.com/string/incr.html#incr) 和 [*EXPIRE*](http://doc.redisfans.com/key/expire.html#expire) 之间存在着一个竞争条件，假如客户端在执行 [*INCR*](http://doc.redisfans.com/string/incr.html#incr) 之后，因为某些原因(比如客户端失败)而忘记设置 [*EXPIRE*](http://doc.redisfans.com/key/expire.html#expire) 的话，那么这个计数器就会一直存在下去，造成每个用户只能访问 10 次，这就是个灾难，要消灭这个实现中的竞争条件，我们可以将它转化为一个 Lua 脚本，并放到 Redis 中运行(这个方法仅限于 Redis 2.6 及以上的版本)：

```C
local current
current = redis.call("incr",KEYS[1])
if tonumber(current) == 1 then
    redis.call("expire",KEYS[1],1)
end
```

还有另一种消灭竞争条件的方法，就是使用 Redis 的列表结构来代替 INCR命令。

```C
FUNCTION LIMIT_API_CALL(ip)
current = LLEN(ip)
IF current > 10 THEN
    ERROR "too many requests per second"
ELSE
    IF EXISTS(ip) == FALSE
        MULTI
            RPUSH(ip,ip)
            EXPIRE(ip,1)
        EXEC
    ELSE
        RPUSHX(ip,ip)
    END
    PERFORM_API_CALL()
END
```



Dubbo 的TPS：

```
class StatItem {

    private String name;
    private long lastResetTime;
    // 单位间隔时间
    private long interval;
    private AtomicInteger token;
    // TPS
    private int rate;

    StatItem(String name, int rate, long interval) {
        this.name = name;
        this.rate = rate;
        this.interval = interval;
        this.lastResetTime = System.currentTimeMillis();
        this.token = new AtomicInteger(rate);
    }

    public boolean isAllowable(URL url, Invocation invocation) {
        long now = System.currentTimeMillis();
        //过了标记的单位时间，直接重置
        if (now > lastResetTime + interval) {
            token.set(rate);
            lastResetTime = now;
        }

        int value = token.get();
        boolean flag = false;
        //尝试获取一个token
        while (value > 0 && !flag) {
            flag = token.compareAndSet(value, value - 1);
            value = token.get();
        }

        return flag;
    }
}
```

​        这里主要用了一个重设时间戳标识，显然Dubbo默认的TPS限流是针对单个JVM而言的，Dubbo做的是通用的，开发者可以自己实现定制限流接口。



#### 2. 滑动窗口

​        滑动窗口（Sliding window）是一种[流量控制](http://baike.baidu.com/view/190232.htm)技术。TCP网络协议用的就是这种算法，上述讲的计数器法有一个明显的问题，就是因为用的自然单位间隔时间，如果在第一分钟的附近集中大量请求，本来应该限流的，但是被分配到了前后两个时间单元，没有限流，这样有些不精确。所以介绍滑动窗口可以解决这个问题，copy过来一张图片：

![](http://i1.piimg.com/567571/c13aad6c92a28892.png)

我们将一分钟分6个10秒，共6个计数单元，比方1:05秒，是落在了1：00 - 1：10那个格子里面，那么统计：0：10 - 1：10的计数器，如果没超过，就在1：00 - 1：10那个格子对应的计数器加一。这有些微分的思想，如果格子越小，那么窗口就越平滑地滑动，显然，计数器法也是滑动窗口的一个特例，如果将1分钟作为一个单元的话。



#### 3. 队列法

​        在计数器法介绍的一分钟内输入三次就出验证码的例子是我当时在优酷土豆实习的时候遇到的，当时想，这个一分钟其实是自然时钟的分钟了，有没有其他方法可以控制成任意时间间隔的一分钟呢，当时借了同事一本马踏飞燕封面的《Redis入门指南》，看到了用队列解决这个问题：

```lua
        listLength = LLEN rate.limiting:IP
        if listLength < 10
            LPUSH rate.limiting:IP, now()
        else
            time = LINDEX rate.limiting:IP, -1
        if now() - time < 60
            ERROR "qps to high, try again later plz"
        else
        LPUSH rate.limiting:IP, now()
        LTRIM rate.limiting:IP, 0, 9
```

这里维护一个size为10的队列，用时间戳来作为值，理想情况下应该是时间戳从小到大入队，这个队列可能存了几分钟前入的结点，当达到了10的大小才去判断和处理。和上面Redis介绍的队列用法对比一下，这里没有设置KEY的过期时间，如何回收，这是一个问题。



#### 4. 漏桶算法

​    漏桶算法，也叫Leaky Bucket，是网络世界中[流量整形](http://baike.baidu.com/view/3344032.htm)（Traffic Shaping）或速率限制（Rate Limiting）时经常使用的一种算法，它的主要目的是控制数据注入到网络的速率，平滑网络上的突发流量。贴一张开涛画的图：

![漏桶](http://p1.bpimg.com/567571/a97718e1b9e8258f.png)

1. 系统以任意速率流入水滴，可以看作请求，桶的容量是容纳水滴的总个数；
2. 如果桶装满了，直接丢弃水滴（请求）；
3. 即使桶内很多水滴，也按照常量流速流出，只能间隔一定时间放走请求。	

伪代码如下：

```java
public class LeakyBucket {
    double rate;               // leak rate in calls/s
    double burst;              // bucket size
    long lastRefreshTime;      // time for last water refresh
    double water;              // water count at refreshTime

    public void refreshWater() {
        long  now = System.currentTimeMillis()/1000;
        //how time flies.
        water = Math.max( 0, water - (now - lastRefreshTime)*rate );
        lastRefreshTime = now;
    }

    public boolean permissionGranted() {
        refreshWater();
        if (water < burst) { // bucket not overflow.
            water ++;
            return true;
        } else {
            return false;
        }
    }
}
```



注意维基百科上对于漏桶算法有两种不同的描述，这是其中的一种，以前写过关于Nginx的限流中提到ngx_http_limit_req_module也是漏桶算法的实现。



#### 5.令牌桶算法

令牌桶也叫Token Bucket，系统按照一定速率往桶内放入token，如果桶满了直接丢弃，请求来了带走一个token直接走，无需等待。也是开涛的图：

![](http://i1.piimg.com/567571/8c55458e52f49589.png)

令牌桶算法和漏桶算法的对比：

1. 令牌桶允许一定程度的突发，而漏桶主要目的是平滑流入速率；
2. 漏桶随意流入请求，固定流出；而令牌桶固定流入token，随意流出请求，请求在有token下无需等待；
3. 漏桶算法和令牌桶算法的最大区别就是令牌桶可以直接一次性拿走多个，而漏桶则就算是空桶状态同时多个请求也要按照固定流速流出，即漏桶是要移除时间的。

​        所以，在某些情况下，漏桶算法不能够有效地使用网络资源。因为漏桶的漏出速率是固定的，所以即使网络中没有发生拥塞，漏桶算法也不能使某一个单独的数据流达到端口速率。因此，漏桶算法对于存在突发特性的流量来说缺乏效率。而令牌桶算法则能够满足这些具有突发特性的流量。



### Guava RateLimiter

Guava的RateLimiter是令牌桶的一种实现，我看的源码是Guava 16的，最新的版本写法上稍有不同。RateLimiter和上述算法比较需要注意的地方：

1. 支持预支系统未来的token。请求的许可数从来不会影响到请求本身的限制（调用acquire(1) 和调用acquire(1000) 将得到相同的限制效果，如果存在这样的调用的话），但会影响下一次请求的限制，也就是说，如果一个高开销的任务抵达一个空闲的RateLimiter，它会被马上许可，但是下一个请求会经历额外的限制，从而来偿付高开销任务。
2. 支持Bursty（突发）和WarmingUp（预热）两种形式；
3. 支持重设QPS，并且如果有积累的token会兑换成相应的大小；
4. 支持阻塞等待（可以设置超时）和立即返回结果两种接口。

自己画的类图：

![](http://p1.bpimg.com/567571/152106997455fbc3.png)

可以看出主要有三种接口，create方法，acquire方法和tryAcquire方法：

1. create方法是创建Bursty对象或者WarmingUp对象；
2. acquire方法是阻塞式获取许可，会一直阻塞等待直到获得许可，返回等待的时间；
3. tryAcquire方法是尝试在一定时间内是否能获得许可，如果不能直接返回false；如果可以，会阻塞等待返回true，注意，可能会阻塞！！！如果不想阻塞，设置超时为0即可。



ticker：是时钟；

offsetNanos：RateLimiter被创建的时间戳，用来避免溢出；

storedPermits： 当前存储的permit个数；

maxPermits：允许存储最大permit数，即桶的容量；

nextFreeTicketMicros：下一次请求（无论其大小）将被授予的时间，在授予一个请求后，这个数会变大，可能是过去或未来；为下次可以获得permit的时间和初始化的时间差

stableIntervalMicros： The interval between two unit requests. 1/qps ms

Bursty：maxBurstSeconds，默认1秒，初始化桶的容量 = QPS * maxBurstSeconds；

其中，初始化设置QPS大小的代码：

```java
  public final void setRate(double permitsPerSecond) {
    Preconditions.checkArgument(permitsPerSecond > 0.0
        && !Double.isNaN(permitsPerSecond), "rate must be positive");
    synchronized (mutex) {
      resync(readSafeMicros());
      double stableIntervalMicros = TimeUnit.SECONDS.toMicros(1L) / permitsPerSecond;
      this.stableIntervalMicros = stableIntervalMicros;
      doSetRate(permitsPerSecond, stableIntervalMicros);
    }
  }
```

readSafeMicros()为现在和初始化的时间差（nowMicros），如果nowMicros > nextFreeTicketMicros(初始值为0)，则将现在与过去的时间差兑换成permit加到storedPermits上，同时刷新nextFreeTicketMicros = nowMicros.
 所以microsToNextFreeTicket必然>=0. 所以只要现在时间>=nextfree时间就可以获得 N permits
 设置stableIntervalMicros=1/qps；接着调用子类Bursty实现的抽象方法doSetRate(qps,stableIntervalMicros)：设置maxPermits，设置storedPermits(如果是刚初始化则赋值storedPermits=0.0)。



WarmingUp:  maxPermits = warmupPeriodMicros / stableIntervalMicros; warmup时间越长，可以存储的permits越多，double availablePermitsAboveHalf = storedPermits - halfPermits;
availablePermitsAboveHalf > 0是要大于时间间隔获得permit的。说明warmup时间越长，获得的越慢，到storedPermits为maxPermits的一半才是正常速率。其中如果长时间不访问，又会回到最初的冷却状态。



### 实现的限流项目

主要用Guava的RateLimiter作为基础来实现，提供了annotation方式和Json配置方式。annotation方式书写方便，Json配置方式可以放到服务配置中心，方便动态更新。

下面是注解方式提供的配置接口：

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface RateLimit {

	double qps();

	/**
	 * 以判断能否在timeout内获得token的方式获取许可。若能则最多阻塞timeout，不能直接返回。
	 * 负数标志不使用timeout方式，而是默认阻塞等待许。
	 * timeout单位是毫秒(1秒=1000毫秒)
	 */
	long timeout() default 0;

	/**
	 * 以预热形式限流。在预热时间内，每秒分配的许可数会平稳地增长直到预热期结束时达到其最大速率。
	 * 默认负数不预热
	 * warmupPeriod单位是毫秒(1秒=1000毫秒)
	 */
	long warmupPeriod() default -1;
```

几乎完全覆盖了RateLimiter提供的接口，其中重设QPS需要自己实现。


