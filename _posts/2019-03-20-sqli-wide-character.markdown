---
layout:     post
title:      "SQL 注入之宽字符注入"
subtitle:   "\\ == 0x5C"
date:       2019-03-20 23:00 +0800
author:     "gsfish"
header-img: "img/post-bg-07.jpg"
tags:
    - Web 安全
    - SQL 注入
    - 渗透
---


# 0x00 MySQL 字符集

通过运行如下语句，可以看到 MySQL 有 8 个与字符集相关的环境变量：

```
MariaDB [(none)]> SHOW VARIABLES LIKE 'character%';
+--------------------------+----------------------------+
| Variable_name            | Value                      |
+--------------------------+----------------------------+
| character_set_client     | gbk                        |
| character_set_connection | gbk                        |
| character_set_database   | gbk                        |
| character_set_filesystem | binary                     |
| character_set_results    | gbk                        |
| character_set_server     | gbk                        |
| character_set_system     | utf8                       |
| character_sets_dir       | /usr/share/mysql/charsets/ |
+--------------------------+----------------------------+
8 rows in set (0.002 sec)
```

其中各个变量的作用如下：

* `character_set_client`：MySQL 服务端认为客户端连接所使用的编码
* `character_set_connection`：没有指定编码时 MySQL 服务端处理所使用的编码
* `character_set_database`：创建数据库时，如果没有指定编码，默认的编码值
* `character_set_results`：查询结果以这个编码发送给客户端
* `character_set_server`：服务器的默认编码，以上所有的变量，默认都和这个设置保持一致

其中 3 个主要变量（`character_set_client`、`character_set_connection`、`character_set_results`）的修改方式如下：

* 非持久化：`SET NAMES 'charset_name' [COLLATE 'collation_name']`
* 持久化：修改 MySQL 服务端配置文件的 `character-set-server`、`skip-character-set-client-handshake` 字段

在处理客户端查询时，MySQL 中字符集的转换过程如下：

1. MySQL 服务端收到请求时，将请求数据从 `character_set_client` 转换为 `character_set_connection`
2. 进行内部操作前，将请求数据从 `character_set_connection` 转换为内部操作字符集，其确定方法如下：
   * 使用每个数据字段的 `CHARACTER SET` 设定值
   * 若上述值不存在，则使用对应数据表的 `DEFAULT CHARACTER SET` 设定值(MySQL 扩展，非 SQL 标准)
   * 若上述值不存在，则使用对应数据库的 `DEFAULT CHARACTER SET` 设定值
   * 若上述值不存在，则使用 `character_set_server` 设定值
3. 将操作结果从内部操作字符集转换为 `character_set_results`

# 0x01 产生原因

首先，MySQL 服务端会根据当前会话的 `character_set_client` 变量解码客户端传来的 SQL 语句。当该变量的值为 `gbk` 时，MySQL 会将符合 GBK 定义的双字节解码为汉字：首字节在 `81-FE` 范围内，尾字节在 `40-FE` 范围内。

然后，假设攻击者使用 `\xdf'` 来逃逸单引号，后端对用户输入进行了转义，单引号 `'` 被 `\`（`\x5c`）转义，形成了 `\xdf\x5c'`。而 `\xdf` 在 `81-FE` 的范围之间，`\x5c` 在 `40-FE` 的范围之间，因此 `\xdf\x5c` 一起被解码为一个汉字，使得 `'` 摆脱了 `\` 的转义。

漏洞产生的场景如下：

1. MySQL `character_set_client` 被配置为 `gbk`，业务使用 `addslashes()` / `mysql_escape_string()` / `magic_quotes_gpc` 转义用户输入
2. 业务通过 `SET NAMES 'gbk'` 将连接字符集设置为 `gbk`，并使用 `addslashes()` / `mysql_escape_string()` / `magic_quotes_gpc` 转义用户输入
3. 业务直接使用 `mysql_real_escape_string()` 转义用户输入，但未通过 `mysql_set_charset()` 设置连接句柄的字符集（`SET NAMES` 无用）

# 0x02 修复方案

1. 使用参数化查询
2. 先使用 `mysql_set_charset()` 设置连接字符集为 `gbk`，再使用 `mysql_real_escape_string()` 来转义用户输入
3. 手动将 `character_set_client` 设置为 `binary`：`SET character_set_connection=gbk, character_set_results=gbk,character_set_client=binary`，再使用相应函数转义用户输入

# 参考资料

1. [phith0n. 浅析白盒审计中的字符编码及SQL注入[EB/OL]. https://www.leavesongs.com/PENETRATION/mutibyte-sql-inject.html](https://www.leavesongs.com/PENETRATION/mutibyte-sql-inject.html)
2. [Laruence. 深入理解SET NAMES和mysql(i)_set_charset的区别[EB/OL]. http://www.laruence.com/2010/04/12/1396.html](http://www.laruence.com/2010/04/12/1396.html)
3. [Laruence. 深入Mysql字符集设置[EB/OL]. http://www.laruence.com/2008/01/05/12.html](http://www.laruence.com/2008/01/05/12.html)
