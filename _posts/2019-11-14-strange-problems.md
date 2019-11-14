---
typora-root-url: ../
layout:     post
title:      各种疑难杂症
subtitle:   用于记录开发与运维过程中遇到的一些坑，希望可以帮助到有需要的人
author:     gsfish
date:       2019-11-14 19:00 +0800
header-img: img/post-bg-07.jpg
tags:
---

## JNI 线程安全解决方案

参考资料：

1. [How do you properly synchronize threads on the native side of a JNI environment](https://stackoverflow.com/questions/44420937/how-do-you-properly-synchronize-threads-on-the-native-side-of-a-jni-environment)
2. [Android JNI 编程实践](https://www.jianshu.com/p/9b83cc5a5ba8)

## ES 过滤聚合结果方案

参考资料：

1. [ElasticSearch 如何先聚合后过滤](https://elasticsearch.cn/question/656)
2. [ES 如何将聚合结果作为条件进行过滤](https://elasticsearch.cn/question/3581)

## PHP 报错：Can't use function return value in write context

原因：

PHP 5.5 以下的版本，只能向 `empty()` 函数传递变量，不能传递引用（函数等）

## MySQL 客户端连接本地服务器时无法指定端口

当目标地址为 `localhost`（默认）时，MySQL 客户端使用 unix socket 进行连接，需改为 `127.0.0.1`

参考资料：

1. https://serverfault.com/questions/306421/why-does-the-mysql-command-line-tool-ignore-the-port-parameter

## Docker for Windows 无法监听 2375 端口

`2375` 在 Windows 的保留范围内，使用前需将其排除

参考资料：

1. https://github.com/docker/for-win/issues/3546

## Windows 安装 mysqlclient 时失败

安装 Visual C++ Build Tools 2015、MySQL Connector C 6.1、MariaDB connector

将 MariaDB 安装目录下相应的文件复制到：

```
C:\Program Files (x86)\MySQL\MySQL Connector C 6.1\include\mariadb
C:\Program Files (x86)\MySQL\MySQL Connector C 6.1\lib\mariadb
```

参考资料：

1. https://devblogs.microsoft.com/python/unable-to-find-vcvarsall-bat/
2. https://stackoverflow.com/questions/51294268/pip-install-mysqlclient-returns-fatal-error-c1083-cannot-open-file-mysql-h

## docker-compose 中的服务需要等待其他服务启动完成后再执行

使用 docker-compose 的 `healthchecks` 命令

参考资料：

1. https://stackoverflow.com/questions/31746182/docker-compose-wait-for-container-x-before-starting-y/41854997#41854997

## 在 windows 运行 celery task 报错：ValueError: not enough values to unpack (expected 3, got 0)

celery 4.x 后不再支持 windows，问题与底层的 billiard 相关（拖了 2 年，曾有人提过修复的 PR，不过作者似乎并不愿意 “修复”）

解决方案（选其一）：

1. 设置环境变量：`set FORKED_BY_MULTIPROCESSING=1`
2. 使用 `- P eventlet/gevent/solo` 运行 celery

参考资料：

1. https://www.distributedpython.com/2018/08/21/celery-4-windows/
2. https://stackoverflow.com/questions/37255548/how-to-run-celery-on-windows/47331438
3. https://github.com/celery/celery/issues/4081
4. https://github.com/celery/celery/issues/4178

# 删除 git 仓库中的子模块

git 本身没有提供相应的命令，可通过以下步骤手动实现：

```
git submodule deinit <path_to_submodule>
git rm --cached <path_to_submodule>
git commit-m "Removed submodule"
```

## 参考资料

1. https://gist.github.com/myusuf3/7f645819ded92bda6677

## 修改 git 仓库中子模块的 url

修改 `.gitmodules` 文件中对应模块的 url 属性

使用 `git submodule sync` 命令，将新的 url 更新到文件 `.git/config`

参考资料：

1. https://www.jianshu.com/p/ed0cb6c75e25

## unitest.patch 无法多次 patch 同一对象

背景：

在 test1 中对导入 func_module.func() 进行测试，func() 引用了 moduleA，对 moduleA 进行 patch 并返回 Mock1，执行 func()

在 test2 中同样导入 func_module.func() 进行测试，func() 引用了 moduleA，重新对对 moduleA 进行 patch 并返回 Mock2，执行 func()

此时 test2 中 func() 内是用的 moduleA 对象是 test1 中的 Mock1，会导致对 Mock2 做的断言失败

原因：

运行 test1 时，对 moduleA 进行了 patch，mock_moduleA 由 func_module 导入并被 func() 使用。运行 test2 时，重新对 moduleA 进行 patch，而由于当前上下文中已存在 moduleA 的导入对象了（test1 时导入的 mock_moduleA），因此解释器没有重新导入，直接使用了 test1 时的 mock_moduleA。究其根本原因，应该是 pytest 对每个 test 的隔离不够充分，使得 test1 中的模块导入链影响了 test2

解决方案：

应改为对 func_module.moduleA 进行 patch（被 func() 直接使用），而非 patch 全局的 moduleA（被 func_module 导入）

参考资料：

1. https://stackoverflow.com/questions/5341147/how-to-patch-a-modules-internal-functions-with-mock

## pyppeteer 无法 setCookie

setCookie 时需指定对应的 url

参考资料：

1. https://github.com/miyakogi/pyppeteer/issues/94

## pyppeteer 访问部分站点时，会关闭与 chromium 的 websockets 连接：错误码 1006

原因：

是 chromium 的 websockets 服务端的问题所导致的，websockets>6 会有一个 ping-pong 的超时判断，若出现超时则会关闭连接

解决方案（选其一）：

1. 在 `pyppeteer/connection.py` 第 44 行为 `websockets.client.connect()` 增加 `ping_interval=None, ping_timeout=None` 参数（websockets>6）
2. 将 websockets 降级至 6.0

参考资料：

1. https://github.com/miyakogi/pyppeteer/issues/62
2. https://github.com/miyakogi/pyppeteer/pull/160
3. https://bugs.chromium.org/p/chromium/issues/detail?id=865002

## pyppeteer 在 scrapy 中同步执行，无法充分利用其异步特性

参考资料：

1. https://github.com/lopuhin/scrapy-pyppeteer
2. https://www.lizenghai.com/archives/24943.html