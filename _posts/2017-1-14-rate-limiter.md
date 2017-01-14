---
layout: post
categories: [架构]
tags: [架构]
code: true
title: 限流方案的设计
---

       记得开涛介绍过，在开发高并发系统时有三把利器用来保护系统：缓存、降级和限流。缓存的目的是提升系统访问速度和增大系统能处理的容量，可谓是抗高并发流量的银弹；而降级是当服务出问题或者影响到核心流程的性能则需要暂时屏蔽掉，待高峰或者问题解决后再打开；而有些场景并不能用缓存和降级来解决，比如稀缺资源（秒杀、抢购）、写服务（如评论、下单）、频繁的复杂查询（评论的最后几页），因此需有一种手段来限制这些场景的并发/请求量，即限流。   

### 算法介绍
#### 1. 计数器法
计数器法是指自然单位时间内统计次数，时间进入下一个单位时间则重置大小，重新计数。这个是最常见项目中用的最多的，比方一分钟内输入密码错误三次则要输入验证码。

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
我看Redis中文文档中将原文“EXPIRE(keyname,10)”写成了“EXPIRE(keyname,1)”，我认为这两种写法会造成限流的结果有一些差异，因为多个JVM的机器的时钟可能会有细微差别（其实因为各个机器时钟的差别我认为这种方式不能真正意义上的精确地控制每一秒的请求数量，因为时间戳是JVM所在机器提供的，而不是Redis提供），而且上述代码时间戳的获得和GET命令的时间可能不是同一秒，可能GET了上一秒的结果，而上一秒如果是跑的最快的那个机器在那秒的开始了执行了“EXPIRE(keyname,1)”，这时候最慢的那台机器去获得的是这个key过期了，再次执行事务会该key会从1开始重新计算，而如果这一秒超过了最大请求次数，显然按照英文文档的做法是不允许通过，而中文文档该key刚好过期了是可以通过的。这里用了事务打包来避免引入竞争条件，还有不用事务的方式：

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

而作者认为在 [*INCR*](http://doc.redisfans.com/string/incr.html#incr) 和 [*EXPIRE*](http://doc.redisfans.com/key/expire.html#expire) 之间存在着一个竞争条件，假如客户端在执行 [*INCR*](http://doc.redisfans.com/string/incr.html#incr) 之后，因为某些原因(比如客户端失败)而忘记设置 [*EXPIRE*](http://doc.redisfans.com/key/expire.html#expire) 的话，那么这个计数器就会一直存在下去，造成每个用户只能访问 10 次，这就是个灾难，要消灭这个实现中的竞争条件，我们可以将它转化为一个 Lua 脚本，并放到 Redis 中运行(这个方法仅限于 Redis 2.6 及以上的版本)：

```C
local current
current = redis.call("incr",KEYS[1])
if tonumber(current) == 1 then
    redis.call("expire",KEYS[1],1)
end
```

还有另一种消灭竞争条件的方法，就是使用 Redis 的列表结构来代替 INCR命令：

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

这里主要用了一个重设时间戳标识，显然Dubbo默认的TPS限流是针对单个JVM而言的，Dubbo做的是通用的，开发者可以自己实现定制限流接口。



#### 2. 滑动窗口

滑动窗口（Sliding window）是一种[流量控制](http://baike.baidu.com/view/190232.htm)技术。TCP网络协议用的就是这种算法，上述讲的计数器法有一个明显的问题，就是因为用的自然单位间隔时间，如果在第一分钟的附近集中大量请求，本来应该限流的，但是被分配到了前后两个时间单元，没有限流，这样有些不精确。所以介绍滑动窗口可以解决这个问题，copy过来一张图片：

![](http://i1.piimg.com/567571/c13aad6c92a28892.png)

我们将一分钟分6个10秒，共6个计数单元，比方1:05秒，是落在了1：00 - 1：10那个格子里面，那么统计：0：10 - 1：10的计数器，如果没超过，就在1：00 - 1：10那个格子对应的计数器加一。显然这有些分治或者微分的思想，如果格子越小，那么窗口就越平滑地滑动，显然，计数器法也是滑动窗口的一个特例，如果将1分钟作为一个单元的话。
