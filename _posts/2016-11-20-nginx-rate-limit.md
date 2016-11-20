---
layout: post
categories: [Nginx]
tags: [Nginx]
code: true
title: nginx限流のlimit_zone与limit_req_zone
---

 

​	Nginx 有多种限流的方式，在Nginx官网文档都有详细描述，可以查阅，下面我总结介绍一下：

### 1.ngx_http_limit_req_module

  限制单个key(如IP)的访问速率，漏桶算法实现。   

```c
http {
    #定义一个名为one的limit_req_zone用来存储session，大小是10M内存，
    #以$binary_remote_addr 为key,限制平均每秒的请求为1个，
    #1M能存储16000个状态，rete的值必须为整数
    limit_req_zone $binary_remote_addr zone=one:10m rate=1r/s;
    ...
    server {
        ...
        location /search/ {
            #可以在后面加上nodelay，6个请求将在第0秒执行。
            limit_req zone=one burst=5;
        }
```



### 2.ngx_http_limit_conn_module

​	限制单个key(如IP)的连接数。

```c
http {
  
    limit_conn_zone $binary_remote_addr zone=addr:10m;
    ...
    server {
        ...
        location /download/ {
          	#每个IP的连接数限制为1
            limit_conn addr 1;
          
          #每秒带宽限制,对单个连接限数，如果一个ip两个连接，就是500Kx2
           limit_rate 500k;  
        }
```

​        在这里使用的是$binary_remote_addr 而不是 $remote_addr。$remote_addr 的长度为 7 至 15 bytes，会话信息的长度为 32 或 64 bytes。 而 $binary_remote_addr 的长度为 4 bytes，会话信息的长度为 32 bytes。 当 zone 的大小为 1M 的时候，大约可以记录 32000 个会话信息（一个会话占用 32 bytes）。

### 3.limit_rate

​	限制向客户端(针对单个连接数)传送响应的速率限制。参数 rate 的单位是字节/秒，设置为 0 将关闭限速。 nginx 按连接限速，所以如果某个客户端同时开启了两个连接，那么客户端的整体速率是这条指令设置值的 2 倍。属于ngx_http_core_module。

```c
server {
    if ($slow) {
        set $limit_rate 4k;
    }
    ...
}
```



### 4.limit_rate_after

​	设置不限速传输的响应大小。当传输量大于此值时，超出部分将限速传送。

```c
location /flv/ {
    flv;
    #3m内不限速，超过部分限速50k/秒
    limit_rate_after 3m;
    limit_rate       50k;
}
```

  以上配置会对所有的ip都进行限制，有些时候我们不希望对搜索引擎的蜘蛛或者自己测试ip进行限制，对于特定的白名单ip我们可以借助geo指令实现。

  再介绍一个测试工具：ab，ab是Apache HTTP server benchmarking tool的简称， 以上配置可以用ab来测试。ab比较常见的参数有以下几个：



```c
//在测试会话中所执行的请求个数（本次测试总共要访问页面的次数）。默认时，仅执行一个请求。
-n requests Number of requests to perform
//一次产生的请求个数（并发数）。默认是一次一个。我理解一次一个的并发是指前一个request已经response了。
-c concurrency Number of multiple requests to make
//测试所进行的最大秒数。其内部隐含值是-n 50000。它可以使对服务器的测试限制在一个固定的总时间以内。默认时，没有时间限制。
-t timelimit Seconds to max. wait for responses
//可以带cookie字符串访问。
-C attribute Add cookie, eg. -C “c1=1234,c2=2,c3=3” (repeatable)
  
-v 设置显示信息的详细程度 - 4或更大值会显示头信息， 3或更大值可以显示响应代码(404, 200等), 2或更大值可以显示警告和其他信息。 -V 显示版本号并退出。

```





参考资料：  

[Nginx官方文档](http://nginx.org/en/docs/)

[nginx限制某个IP同一时间段的访问次数](http://www.nginx.cn/446.html) 

​    