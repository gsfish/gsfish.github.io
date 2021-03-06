---
typora-root-url: ../
layout:     post
title:      智能卡安全之 M1 卡破解
subtitle:   入门篇，不知道能不能出系列
author:     gsfish
date:       2018-10-30 21:30 +0800
header-img: img/post-bg-hacker.jpg
tags:
 - 无线安全
---


> 之所以起这个标题，是想有朝一日可以写成一个系列。由于目前手里只有一张未加密的 M1（MIFARE Classic 1K）卡，因此本文仅包含对 M1 卡的破解。


# 0x00 智能卡概况

M1 卡可分为 16 个扇区，每个扇区可分为 4 个段，每个段为 16 字节（即 `1K = 16 * 4 * 16B`）。

其中每个扇区的最后一段包含独立密钥，布局为：Key A(6B) + 控制位(4B) + Key B(6B)

M1 卡的整体布局如下：

```
00000000: 56b1 ce83 aa08 0400 6263 6465 6667 6869  V.......bcdefghi
00000010: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000020: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000030: ffff ffff ffff ff07 8069 ffff ffff ffff  .........i......
00000040: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000050: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000060: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000070: ffff ffff ffff ff07 8069 ffff ffff ffff  .........i......
00000080: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000090: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000000a0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000000b0: ffff ffff ffff ff07 8069 ffff ffff ffff  .........i......
000000c0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000000d0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000000e0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000000f0: ffff ffff ffff ff07 8069 ffff ffff ffff  .........i......
00000100: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000110: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000120: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000130: ffff ffff ffff ff07 8069 ffff ffff ffff  .........i......
00000140: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000150: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000160: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000170: ffff ffff ffff ff07 8069 ffff ffff ffff  .........i......
00000180: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000190: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000001a0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000001b0: ffff ffff ffff ff07 8069 ffff ffff ffff  .........i......
000001c0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000001d0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000001e0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000001f0: ffff ffff ffff ff07 8069 ffff ffff ffff  .........i......
00000200: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000210: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000220: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000230: ffff ffff ffff ff07 8069 ffff ffff ffff  .........i......
00000240: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000250: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000260: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000270: ffff ffff ffff ff07 8069 ffff ffff ffff  .........i......
00000280: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000290: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000002a0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000002b0: ffff ffff ffff ff07 8069 ffff ffff ffff  .........i......
000002c0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000002d0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000002e0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000002f0: ffff ffff ffff ff07 8069 ffff ffff ffff  .........i......
00000300: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000310: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000320: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000330: ffff ffff ffff ff07 8069 ffff ffff ffff  .........i......
00000340: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000350: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000360: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000370: ffff ffff ffff ff07 8069 ffff ffff ffff  .........i......
00000380: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000390: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000003a0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000003b0: ffff ffff ffff ff07 8069 ffff ffff ffff  .........i......
000003c0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000003d0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000003e0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
000003f0: ffff ffff ffff ff07 8069 ffff ffff ffff  .........i......
```

我们从淘宝上可以买到的白卡大致分为一下几种：

| 类型  | 说明                                                     |
| ----- | -------------------------------------------------------- |
| M1    | 普通 IC 卡，除 0 扇区不可修改，其他扇区可重复擦写        |
| UID   | 普通 IC 复制卡（Chinese clones），可重复擦写所有扇区     |
| CUID  | 用于绕过系统对 UID 卡的拦截                              |
| FUID  | 用于绕过系统对 CUID 卡的拦截，0 扇区写入一次后变成 M1 卡 |
| UFUID | UID 和 FUID 的合成卡，需要封卡操作                       |


# 0x01 工具准备

出于省钱的考虑采用的方案为：NXP 自家的 PN532 NFC/RFID v3 模块 + CH340 USB 转 TTL 模块

硬件：

1. PN532
2. CH340
3. UID 白卡若干

软件：

1. libnfc
2. mfoc
3. mfcuk

M1（MIFARE Classic 1K）卡是我们日常生活中经常接触到的智能卡之一，本次破解为使用 UID 卡对博主小区的 M1 门禁卡进行复制（小区门禁是检测 0 扇区的数据）。

由于 Linux 内核自 2.6 之后默认添加了对 CH340/CH341 芯片的驱动支持，将 PN532 连接好，`libnfc` 安装完成后，即可使用 `nfc-list` 检测系统是否成功识别 NFC 设备。若无法识别可尝试在 `/etc/nfc/libnfc.conf` 中设置如下选项以自动检测设备：

```
allow_autoscan = true
allow_intrusive_scan = true
```


# 0x02 破解扇区密钥

首先使用 `mfoc` 尝试导出扇区转储：

```
sudo mfoc -O mycard.mfd
```

该工具会使用内嵌的默认密码字典尝试解密各扇区，若出现如下输出则说明该卡需要破解：

```
Try to authenticate to all sectors with default keys...
Symbols: '.' no key found, '/' A key found, '\' B key found, 'x' both keys found
[Key: ffffffffffff] -> [................]
[Key: a0a1a2a3a4a5] -> [................]
[Key: d3f7d3f7d3f7] -> [................]
[Key: 000000000000] -> [................]
[Key: b0b1b2b3b4b5] -> [................]
[Key: 4d3a99c351dd] -> [................]
[Key: 1a982c7e459a] -> [................]
[Key: aabbccddeeff] -> [................]
[Key: 714c5c886e97] -> [................]
[Key: 587ee5f9350f] -> [................]
[Key: a0478cc39091] -> [................]
[Key: 533cb6c723f6] -> [................]
[Key: 8fd0a4f256e9] -> [................]
 
Sector 00 -  UNKNOWN_KEY [A]  Sector 00 -  UNKNOWN_KEY [B]  
Sector 01 -  UNKNOWN_KEY [A]  Sector 01 -  UNKNOWN_KEY [B]  
Sector 02 -  UNKNOWN_KEY [A]  Sector 02 -  UNKNOWN_KEY [B]  
Sector 03 -  UNKNOWN_KEY [A]  Sector 03 -  UNKNOWN_KEY [B]  
Sector 04 -  UNKNOWN_KEY [A]  Sector 04 -  UNKNOWN_KEY [B]  
Sector 05 -  UNKNOWN_KEY [A]  Sector 05 -  UNKNOWN_KEY [B]  
Sector 06 -  UNKNOWN_KEY [A]  Sector 06 -  UNKNOWN_KEY [B]  
Sector 07 -  UNKNOWN_KEY [A]  Sector 07 -  UNKNOWN_KEY [B]  
Sector 08 -  UNKNOWN_KEY [A]  Sector 08 -  UNKNOWN_KEY [B]  
Sector 09 -  UNKNOWN_KEY [A]  Sector 09 -  UNKNOWN_KEY [B]  
Sector 10 -  UNKNOWN_KEY [A]  Sector 10 -  UNKNOWN_KEY [B]  
Sector 11 -  UNKNOWN_KEY [A]  Sector 11 -  UNKNOWN_KEY [B]  
Sector 12 -  UNKNOWN_KEY [A]  Sector 12 -  UNKNOWN_KEY [B]  
Sector 13 -  UNKNOWN_KEY [A]  Sector 13 -  UNKNOWN_KEY [B]  
Sector 14 -  UNKNOWN_KEY [A]  Sector 14 -  UNKNOWN_KEY [B]  
Sector 15 -  UNKNOWN_KEY [A]  Sector 15 -  UNKNOWN_KEY [B]  
```

MFCUK（MiFare Classic Universal toolKit） 是一款基于 darkside 攻击原理破解全加密 M1 卡的开源软件，`mfcuk` 通过算法的漏洞破解出第一个 Key，如果某个扇区的 Key 被破解出来，就可以再使用 `mfoc` 工具使用 nested authentication 攻击破解其他扇区的密码：

```
sudo mfcuk -C -R {} -s 250 -S 250

说明：
{} 替换为待破解的扇区号，也可使用切片指定范围 0:A
```


# 0x03 导出扇区转储

使用 `mfoc` 尝试以默认密钥导出扇区转储：

```
sudo mfoc -O mycard.mfd
```

# 0x04 写入新卡

使用 `nfc-mfclassic`（`libnfc` 内置）将转储文件写入白卡：

```
sudo nfc-mfclassic W a mycard.mfd mycard.mfd f

说明：
W 为可写入 0 扇区（UID 卡）
a 为使用 Key A 解密，遇错时退出
第一个 mycard.mfd 为待写入的转储文件
第二个 mycard.mfd f 为包含密钥的转储文件
```

# 参考资料

1. https://mp.weixin.qq.com/s?__biz=MzI5MDQ2NjExOQ==&mid=2247488354&idx=1&sn=be1205869e1be6bf5df3b3f2132d2969
2. https://fanzheng.org/archives/30
3. https://zohead.com/archives/copy-mifare-classic/
