---
typora-root-url: ../
layout:     post
title:      关于 Python 中主线程响应 SIGINT 解决方案
subtitle:   由于主线程开小差而引发的一个问题
author:     gsfish
date:       2017-03-27 12:56 +0800
header-img: img/post-bg-python.jpg
tags:
 - 并发
 - Python
---

# 0x00 引子

最近在 Freebuf 上的[一篇文章](http://www.freebuf.com/sectool/129224.html)的启发下用多线程改写了一个基于 Python 的 Struts2 S2-045 [批量检测工具](https://github.com/gsfish/S2-Reaper)，使用的是 `threading` 模块。`thread` 和 `threading` 都允许程序员创建和管理线程，而 `threading` 对原生的 `thread` 做了进一步的封装，提供了更高级别，更强的线程管理的功能。

在这个项目中：

* 线程 1 为 Google 爬虫，将抓取以关键词搜索到的有关 URL。
* 线程 2 为 PoC 检测，会将具有漏洞的 URL 进一步筛选并执行 Exploit。

然而在实现的过程中遇到了一个问题，觉得挺有意思的：

1. 若将子线程设置为守护线程，则主线程在创建完两个子线程后，由于执行完毕将会退出。
2. 若子线程非守护线程，则主线程在执行完后将会阻塞，直到子线程执行结束。

问题在于，在运行了一段时间后想要退出，当按下 `Control + C` 时由于主线程被子线程阻塞了，因此无法响应终端发出的 SIGINT 信号，所以只好简单粗暴地将整个进程 `kill` 掉。后来在网上收集了一些类似问题的解决方案。


# 0x01 方案一

只有把子线程设成守护线程才能让主线程不等待，以接收 SIGINT 信号，但是又不能让子线程立即结束。因此可以采用轮询的方式在主线程中不断检查子线程是否结束，并使用 `sleep()` 节省 CPU 资源。此方案效率较低，且会产生延迟。

```python
while True:
    alive = False
    for thread in threads:
        alive = alive or thread.is_alive()
    if alive:
        time.sleep(1)
    else:
        break
```


# 0x02 方案二

创建一个新进程来接收 SIGINT 并将执行任务的进程杀掉。其中 `Watch()` 需要在子线程创建前调用。

```python
class Watcher:   
    """this class solves two problems with multithreaded  
    programs in Python, (1) a signal might be delivered  
    to any thread (which is just a malfeature) and (2) if  
    the thread that gets the signal is waiting, the signal  
    is ignored (which is a bug).  
 
    The watcher is a concurrent process (not thread) that  
    waits for a signal and the process that contains the  
    threads.  See Appendix A of The Little Book of Semaphores.  
    http://greenteapress.com/semaphores/  
 
    I have only tested this on Linux.  I would expect it to  
    work on the Macintosh and not work on Windows.  
    """  
  
    def __init__(self):   
        """ Creates a child thread, which returns.  The parent  
            thread waits for a KeyboardInterrupt and then kills  
            the child thread.  
        """  
        self.child = os.fork()   
        if self.child == 0:   
            return  
        else:   
            self.watch()   
  
    def watch(self):   
        try:   
            os.wait()   
        except KeyboardInterrupt:   
            # I put the capital B in KeyBoardInterrupt so I can   
            # tell when the Watcher gets the SIGINT   
            print 'KeyBoardInterrupt'  
            self.kill()   
        sys.exit()   
  
    def kill(self):   
        try:   
            os.kill(self.child, signal.SIGKILL)   
        except OSError: pass
```


# 0x03 方案三

设置全局变量标记子线程是否继续，并在主线程阻塞前处理 `KeyboardInterrupt` 异常。有关代码如下：

```python
import time
import random
from threading import Thread

stop = False
threads_num = 20

todos = list(range(1000))
total = len(todos)

def test(name):
    while todos:
        todo = todos.pop()
        # print('{}获取到 todo-{}'.format(name, todo))
        sleep_time = random.randint(1, 5) / 10
        # print('{}休息{}秒'.format(name, sleep_time))
        time.sleep(sleep_time)
        if stop:
            print('{}收到结束信号正在处理'.format(name))
            break
    print('{}结束'.format(name))


if __name__ == '__main__':
    start_time = time.time()
    # 启动线程
    threads = []
    for i in range(threads_num):
        t = Thread(target = test, args = ('线程-{}'.format(i),))
        threads.append(t)
        t.start()
    
    # 响应 ctrl+c
    try:
        while todos:
            print('已完成{}中的{}，还剩余{}'.format(total, total - len(todos), len(todos)))
            time.sleep(1)
    except KeyboardInterrupt as e:
        print('收到结束信号，正在处理')
        stop = True

    # 确认所有子线程结束
    for t in threads:
        t.join()
        
    print('所有子线程已结束')
    print('执行清理工作...')
    print('共计用时{}秒'.format(time.time() - start_time))
```

0x04 最终方案

最后一想，其实貌似没必要这么麻烦。于是把 PoC 和 Exploit 设置为守护线程，并直接将爬虫作为主线程。


# 参考资料

1. http://www.jb51.net/article/35165.htm
2. http://blog.csdn.net/ace_fei/article/details/8899333
3. https://www.v2ex.com/t/323676
