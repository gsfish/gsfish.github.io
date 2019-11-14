---
typora-root-url: ../img
layout:     post
title:      "某科学上网插件分析"
subtitle:   "穿越_v2.1"
date:       2017-09-04 15:33 +0800
author:     "gsfish"
header-img: "img/post-bg-07.jpg"
tags:
    - 逆向
---


# 0x00 引子

今天一个朋友发过来一个 Chrome 插件，说是花了 39 块钱买的，装到 Chrome 里就能直接翻墙，让帮忙看看是否靠谱。

![01.png](/img/proxy-plugin-analysis/01.png)

# 0x01 分析过程

拿到的是一个 crx 文件，其实就是一个特殊的压缩文件。crx 的逆向非常简单，直接将后缀改为 `.zip` 并解压即可得到所有源码。

插件的主要逻辑代码在 `js` 文件夹下，里面只有四个文件：

```
background.js
bootstrap.min.js
bootstrap-switch.min.js
j.js
```

`bootstrap` 是用来构成 UI 的，涉及插件逻辑的代码应该是在 `background.js` 和 `j.js` 中。这两个 JS 文件只是简单压缩了一下，并没有进行混淆。用在线的 JS 美化工具 [Online JavaScript beautifier](http://jsbeautifier.org/) 美化一下即可。

`background.js` 内容如下：

![02.png](/img/proxy-plugin-analysis/02.png)

这里出现了 `startProxy()` 函数，去另一个文件里找一下。

`j.js` 内容如下：

![03.png](/img/proxy-plugin-analysis/03.png)

可见 `startProxy()` 函数使用 Chrome 的 API 对浏览器代理进行了设置，其中所有的代理服务器都保存在了 `cc` 当中。直接拿到了 4 个 HTTPS 代理，除了科学上网还可以用来做其他事了。这些代理的地址没有采用从服务器获取的方式，而是硬编码在插件当中的。因此在代理失效后只能联系开发者来获取新的浏览器插件。

# 0x02 后续进展

这个插件的运营模式大概是通过向开发者转账来获取最新版本的插件，每个版本 39 元，有效时间未知。到目前为止其中一个代理已经失效了，因此长期使用的话并不推荐。
