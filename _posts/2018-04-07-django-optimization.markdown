---
layout:     post
title:      "Django 性能优化实践"
subtitle:   ""
date:       2018-04-07 21:30 +0800
author:     "gsfish"
header-img: "img/post-bg-python.jpg"
tags:
    - Django
    - Python
---


# 0x00 运行环境

| 运行环境 | 环境配置                                    |
| -------- | ------------------------------------------- |
| 操作系统 | Ubuntu 16.04.3 LTS                          |
| CPU      | Intel(R) Xeon(R) CPU E5-26xx v3 双核 2.0GHz |
| 内存     | 2G                                          |
| 编程语言 | Python 3.5.2                                |
| 后台框架 | Django 2.0.2                                |
| 压测工具 | ApacheBench 2.3                             |


# 0x01 并发访问

## runserver

由于 Django 并没有提供高性能的 server 端来处理连接，所以一般不建议使用该命令在生产环境中部署。

## Gunicorn

Django 在[官方文档](http://docs.gunicorn.org/en/latest/design.html#how-many-workers)中提到过 Worker 数的一个推荐设置 `(2 x $num_cores) + 1`。因此采用了以下方案，使用 Nginx 反向代理（项目的 Web API 仍会用到一些静态资源），Django 负责实现 WSGI，并由 Gunicorn pre-fork 出 5 个 Worker，每个 Worker 通过 gevent 异步处理事务：

nginx + gunicorn (gevent, 5 worker)

## 压力测试

ApacheBench：

ab 命令会创建很多的并发访问线程，模拟多个访问者同时对某一 URL 地址进行访问。它的测试目标是基于 URL 的，因此，既可以用来测试 Apache 的负载压力，也可以测试 nginx、lighthttp、tomcat、IIS 等其它 Web Server的压力。

ab 命令对发出负载的计算机要求很低，既不会占用很高 CPU，也不会占用很多内存，但却会给目标服务器造成巨大的负载，其原理类似 CC 攻击。自己测试使用也须注意，否则一次上太多的负载，可能造成目标服务器因资源耗完，严重时甚至导致死机。

测试对象为项目中涉及 IO 操作较多的一个面板展示接口。为了降低网络延迟对测试结果的影响，采用对本地地址 `127.0.0.1` 压测的方式进行。另外由于软件本身的限制，请求数最大只能为 50000，因此两次测试耗时各不相同。

```
ab -t 60 -c 100 -p params.txt -m post -T application/x-www-form-urlencoded -v 1 http://127.0.0.1:8000/api/v1/dashboard/
```

使用 runserver：

```
Server Software:        WSGIServer/0.2
Server Hostname:        127.0.0.1
Server Port:            8000

# 测试的页面
Document Path:          /api/v1/dashboard/
# 页面大小
Document Length:        1748 bytes

# 测试的并发数
Concurrency Level:      100
# 整个测试持续的时间
Time taken for tests:   60.075 seconds
# 完成的请求数量
Complete requests:      1071
# 失败的请求数量
Failed requests:        0
# 整个过程中的网络传输量
Total transferred:      2108891 bytes
# 整个过程中的 HTML 内容传输量
Total body sent:        266444
HTML transferred:       1872108 bytes
# 最重要的指标之一，相当于 LR 中的每秒事务数，后面括号中的 mean 表示这是一个平均值
Requests per second:    17.83 [#/sec] (mean)
# 最重要的指标之二，相当于 LR 中的平均事务响应时间，后面括号中的 mean 表示这是一个平均值
Time per request:       5609.283 [ms] (mean)
# 每个连接请求实际运行时间的平均值
# 对于并发请求，CPU 实际上并不是同时处理的，而是按照每个请求获得的时间片逐个轮转处理的，所以基本上第一个 Time per request 时间约等于第二个 Time per request 时间乘以并发请求数。
Time per request:       56.093 [ms] (mean, across all concurrent requests)
# 平均每秒网络上的流量，可以帮助排除是否存在网络流量过大导致响应时间延长的问题
Transfer rate:          34.28 [Kbytes/sec] received
                        4.33 kb/s sent
                        38.61 kb/s total

# 网络上消耗的时间的分解，各项数据的具体算法还不是很清楚
Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0 1458 3611.3      0   31076
Processing:  1563 2536 1105.4   2372   29195
Waiting:     1545 2499 1104.0   2335   29174
Total:       1563 3995 3820.4   2901   36210

# 整个场景中所有请求的响应情况，为响应时间的一个分布率。其中 50％ 的用户响应时间小于 2901ms，66％ 的用户响应时间小于 3409ms，最大的响应时间小于 36210ms
Percentage of the requests served within a certain time (ms)
  50%   2901
  66%   3409
  75%   3872
  80%   4661
  90%   5674
  95%   9734
  98%  17420
  99%  17766
 100%  36210 (longest request)
```

使用 Gunicorn：

```
Server Software:        nginx/1.10.3
Server Hostname:        127.0.0.1
Server Port:            8000

# 测试的页面
Document Path:          /api/v1/dashboard/
# 页面大小
Document Length:        182 bytes

# 测试的并发数
Concurrency Level:      100
# 整个测试持续的时间
Time taken for tests:   2.219 seconds
# 完成的请求数量
Complete requests:      50000
# 失败的请求数量
Failed requests:        0
Non-2xx responses:      50000
# 整个过程中的网络传输量
Total transferred:      17150000 bytes
# 整个过程中的 HTML 内容传输量
Total body sent:        11800000
HTML transferred:       9100000 bytes
# 最重要的指标之一，相当于 LR 中的每秒事务数
Requests per second:    22532.28 [#/sec] (mean)
# 最重要的指标之二，相当于 LR 中的平均事务响应时间
Time per request:       4.438 [ms] (mean)
# 每个连接请求实际运行时间的平均值
Time per request:       0.044 [ms] (mean, across all concurrent requests)
# 平均每秒网络上的流量，可以帮助排除是否存在网络流量过大导致响应时间延长的问题
Transfer rate:          7547.43 [Kbytes/sec] received
                        5192.99 kb/s sent
                        12740.42 kb/s total

# 网络上消耗的时间的分解，各项数据的具体算法还不是很清楚
Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    2   0.5      2       7
Processing:     1    2   0.7      2       9
Waiting:        0    2   0.6      2       9
Total:          2    4   0.8      4      11

# 整个场景中所有请求的响应情况，为响应时间的一个分布率
Percentage of the requests served within a certain time (ms)
  50%      4
  66%      4
  75%      4
  80%      4
  90%      5
  95%      5
  98%      8
  99%      9
 100%     11 (longest request)
```

## 小结

根据 `Requests per second` 可以见得，使用 Gunicorn (22532.28) 后在每秒的事务处理量上是默认 runserver (17.83) 的约 1264 倍。


# 0x02 查询优化

// TODO


# 0x03 进一步优化

## 使用 Redis 进行缓存

// TODO

## 使用异步 Worker 进行写库操作

// TODO


# 参考文献

1. [xianglong. Django运行方式及处理流程总结[EB/OL]. http://python.jobbole.com/80836/](http://python.jobbole.com/80836/)
2. [小屋子大侠. Django源码分析2:本地运行runserver分析[EB/OL]. https://blog.csdn.net/qq_33339479/article/details/78873786](https://blog.csdn.net/qq_33339479/article/details/78873786)
3. [Henry Z. Django 性能优化官方文档笔记(主要针对ORM)[EB/OL]. https://changchen.me/blog/20170503/django-performance-and-optimisation/](https://changchen.me/blog/20170503/django-performance-and-optimisation/)
