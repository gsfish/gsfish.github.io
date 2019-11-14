---
typora-root-url: ../
layout:     post
title:      LVM 容量调整日志
subtitle:   记一次因容量不足而引起的 LVM 调整
author:     gsfish
date:       2018-11-05 00:00 +0800
header-img: img/post-bg-07.jpg
tags:
 - Linux
---


# 0x00 前言

博主目前的磁盘分区采用的是 LVM-on-LUKS 方案，布局如下：

```
+-----------------------------------------------------------------------+ +----------------+
| Logical volume 1      | Logical volume 2      | Logical volume 3      | | Boot partition |
|                       |                       |                       | |                |
| [SWAP]                | /                     | /home                 | | /boot          |
|                       |                       |                       | |                |
| /dev/MyVol/swap       | /dev/MyVol/root       | /dev/MyVol/home       | |                |
|_ _ _ _ _ _ _ _ _ _ _ _|_ _ _ _ _ _ _ _ _ _ _ _|_ _ _ _ _ _ _ _ _ _ _ _| |                |
|                                                                       | |                |
|                         LUKS encrypted partition                      | |                |
|                           /dev/sdb2                                   | | /dev/sdb1      |
+-----------------------------------------------------------------------+ +----------------+
```

其中：

* `/dev/mapper/cryptolvm` 为使用 `LUKS` 在 `/dev/sdb2` 创建，并写入了 LVM 的特殊头部的物理卷（PV）
* `/dev/MyVol` 为卷组（VG）
* `/dev/MyVol/{swap,root,home}` 为逻辑卷（LV）

由于 `root` 分区目前只分配了 40G（已调整多次），使用 `pacman` 更新系统时提示剩余空间不够（即使清理了所有软件包缓存…）：

```
错误：分区 / 过满：需要 795917 个块，剩余 345532 个块
错误：无法提交处理 (剩余空间不够)
发生错误，没有软件包被更新。
```


# 0x01 方案一：分别调整 FS 与 LV

网上大部分关于 LVM 逻辑卷（LV） resizing 的文章都是按照下面这个步骤来的，由于缩减文件系统容量时需要指定具体大小，实施起来不是很灵活：

打开加密卷：

```
cryptsetup luksOpen /dev/sdb2 cryptolvm
```

缩容：

1. 使用 `resize2fs` 缩减逻辑卷 A 中文件系统的容量
2. 可选：使用 `e2fsck` 检查文件系统（排错）
3. 使用 `lvreduce` / `lvresize` 缩减逻辑卷 A 的容量

```
resize2fs /dev/MyVol/home <NewSize>
e2fsck -f /dev/MyVol/home
lvreduce -L -10G /dev/MyVol/home
```

扩容：

1. 使用 `lvextend` / `lvresize` 扩大逻辑卷 B 的容量
2. 使用 `resize2fs` 扩大逻辑卷 B 中文件系统的容量
3. 可选：使用 `e2fsck` 检查文件系统（排错）

```
lvextend -l +100%FREE /dev/MyVol/root
resize2fs /dev/MyVol/root
e2fsck -f /dev/MyVol/root
```


# 0x02 方案二：一次性调整 LV

根据 ArchWiki，`lvresize` 的 `-r` / `--resizefs` 选项将文件系统调整、LV 调整二者结合了起来，因此使用该种方式调整容量将更加方便。

打开加密卷：

```
cryptsetup luksOpen /dev/sdb2 cryptolvm
```

缩容：

```
lvresize -L -10G -r /dev/MyVol/home
```

扩容：

```
lvresize -l +100%FREE -r /dev/MyVol/root
```


# 参考资料

1. https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#LVM_on_LUKS
2. https://wiki.archlinux.org/index.php/LVM#Resizing_volumes
3. https://wiki.archlinux.org/index.php/Resizing_LVM-on-LUKS
