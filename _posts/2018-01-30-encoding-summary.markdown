---
layout:     post
title:      "字符编码学习总结"
subtitle:   "Humans use text. Computers speak bytes."
date:       2018-01-30 18:30 +0800
author:     "gsfish"
header-img: "img/post-bg-07.jpg"
tags:
    - 开发
---

有几个概念容易混淆：

* `Unicode`：指的是字符集。其为每一个文字符号分配了唯一的标识
* `UTF-8、UTF-16`：指的是编码方式。其将字符集中的文字标识编码为用于传输或存储的字节序列
* `GBK、GB2312`：为国家规范。既包含了中文字符集，同时也包含了相应的编码方式

# Unicode

Unicode 对世界上大部分的文字系统进行了整理、编码，使得电脑可以用更为简单的方式来呈现和处理文字。

对于 Unicode 中的每个字符，Unicode 为其分配了唯一的字符标识，即码位（Code Point），在 Unicode 标准中以前缀 `U+` 表示，Unicode 中定义的区域为 `U+0000 ~ U+10FFFF`（两到三个字节）。

Unicode 的实现方式不同于编码方式。一个字符的 Unicode 编码是确定的。但是在实际传输过程中，由于不同系统平台的设计不一定一致，以及出于节省空间的目的，对 Unicode 编码的实现方式有所不同。Unicode 的实现方式称为 Unicode 转换格式（Unicode Transformation Format，简称为 UTF）。

# UTF-8

UTF-8 是一种针对 Unicode 的，码元长度为 8 比特的变长编码方式。`U+0080` 的以下字符都使用内含其字符的单字节编码，这些编码正好对应 7 比特的 ASCII 字符。

| 代码范围        | UTF-8                              |
| --------------- | ---------------------------------- |
| 000000 - 00007F | 0zzzzzzz (00-7F)                   |
| 000080 - 0007FF | 110yyyyy (C0-DF) 10zzzzzz (80-BF)  |
| 000800 - 00FFFF | 1110xxxx (E0-EF) 10yyyyyy 10zzzzzz |

UTF-8 以字节为编码单元，它的字节顺序在所有系统中都是一様的，没有字节序的问题，也因此它实际上并不需要 BOM。

# UTF-16

UTF-16 也是一种变长编码方式，不过其码元长度为 16，且无法兼容 ASCII。UTF-16 的编码方式相对来说就比较复杂了，由于其码元的长度为双字节，所以涉及到字节序的问题。也因此，UTF-16 分为 UTF-16LE 和 UTF-16BE。

在 UTF-16 文件的开头，都会放置一个 `U+FEFF` 字符作为字节序标识（UTF-16LE 以 `FF FE` 代表，UTF-16BE 以 `FE FF` 代表）。其中 `U+FEFF` 字符在 Unicode 中代表的意义是 `ZERO WIDTH NO-BREAK SPACE`，顾名思义，它是个没有宽度也没有断字的空白。


| 字符    | 普通二进制               | UTF-16                                  |
| ------- | ------------------------ | --------------------------------------- |
| U+0024  | 0000 0000 0010 0100      | 0000 0000 0010 0100                     |
| U+10437 | 0001 0000 0100 0011 0111 | 1101 1000 0000 0001 1101 1100 0011 0111 |

# BOM

BOM 即为字节顺序标记（Byte-Order Mark），是位于码位 `U+FEFF` 的 Unicode 字符的名称。其一般出现与文件的开头，用于表明文件内容的字节序。

| 编码                 | 表示（十六进制）                                   | 表示（十进制）                                      |
| -------------------- | -------------------------------------------------- | --------------------------------------------------- |
| UTF-8                | EF BB BF                                           | 239 187 191                                         |
| UTF-16BE             | FE FF                                              | 254 255                                             |
| UTF-16LE             | FF FE                                              | 255 254                                             |
| UTF-32BE             | 00 00 FE FF                                        | 0 0 254 255                                         |
| UTF-32LE             | FF FE 00 00                                        | 255 254 0 0                                         |
| UTF-7                | 2B 2F 76和以下的一个字节：[ 38 \| 39 \| 2B \| 2F ] | 43 47 118和以下的一个字节：[ 56 \| 57 \| 43 \| 47 ] |
| UTF-1                | F7 64 4C                                           | 247 100 76                                          |
| UTF-EBCDIC           | DD 73 66 73                                        | 221 115 102 115                                     |
| Unicode 标准压缩方案 | 0E FE FF                                           | 14 254 255                                          |
| BOCU-1               | FB EE 28 及可能跟随着 FF                           | 251 238 40 及可能跟随着 255                         |
| GB-18030             | 84 31 95 33                                        | 132 49 149 51                                       |

# Python 中的编码问题

Python 中常见编码的问题主要有三个，分别为 `UnicodeEncodeError`、`UnicodeDecodeError` 和 `SyntaxError`。

1. `UnicodeEncodeError` 发生在将字符串转换为二进制序列的过程中。可以在 `.encode()` 方法中指定 `errors` 参数的方式来指定当前编码方式对未知字符该如何处理（`ignore`、`replace`、`xmlcharrefreplace`）。

2. `UnicodeDecodeError` 发生在将二进制序列转换为字符串的过程中。同样何在 `.decode()` 方法中指定 `errors` 参数来指定当前解码方式对未知字节的处理方式。也可使用 Python 的编码侦测库 [Chardet](https://pypi.python.org/pypi/chardet) 来检测未知的字节序列所采用的编码方式（仅针对部分流行编码）。

3. `SyntaxError` 主要发生在源码的编码与预期不符，可通过在代码开始处指定编码方式解决：

```python
# coding: utf-8
# 或
# -*- coding: utf-8 -*-
```

### 编码默认值

《流畅的 Python》中有这样一段代码，可用于查看当前编码的默认值：

```python
import sys, locale

expressions = """
    locale.getpreferredencoding()
    type(my_file)
    my_file.encoding
    sys.stdout.isatty()
    sys.stdout.encoding
    sys.stdin.isatty()
    sys.stdin.encoding
    sys.stderr.isatty()
    sys.stderr.encoding
    sys.getdefaultencoding()
    sys.getfilesystemencoding()
"""

my_file = open('dummy', 'w')
for expression in expressions.split():
    value = eval(expression)
    print(expression.rjust(30), '->', repr(value))
```

其中 `locale.getpreferredencoding()` 的返回值最重要，其表示打开文件的默认编码，也是重定向到文件的 `sys.stdout/stdin/stderr` 的默认编码。

`sys.getfilesystemencoding()` 用于编解码文件名（非文件内容），将最终的字节序列传递给系统 API。

`sys.getdefaultencoding()` 用于 Python 内部字节序列与字符串的转换。

在 Linux 中默认编码方式一般都为 UTF-8，而在 Windows 则不同，也因此在 Windows 中更容易遇到编码问题。

### Unicode 三明治

处理文本的最佳实践是“Unicode 三明治”。对输入来说，要尽早把输入的字节序列解码成字符串。在程序的业务逻辑中只能处理字符串对象。对输出来说，要尽量晚地把字符串编码成字节序列。依赖默认编码可能会遇到麻烦，因此最好显示地指定编码方式。

# Windows API 中的编码问题

曾经，Windows NT 面对国际化的需求采用了 UTF-16 作为系统字符编码（[据说当时还没有 UTF-8](https://stackoverflow.com/questions/2995111/why-isnt-utf-8-allowed-as-the-ansi-code-page#answer-2995131)）。

因此，大部分的 Windows API 都有两种版本，一种是 ANSI 版，一种是宽字符版：

1. `...A()`：接受 char 类型参数
2. `...W()`：接受 wchar_t 类型参数（UTF-16）

以 `MessageBox()` 为例：

```c
int MessageBoxA(HWND hWnd, const char* lpText, const char* lpCaption, unsigned int uType);
int MessageBoxW(HWND hWnd, const wchar_t* lpText, const wchar_t* lpCaption, unsigned int uType);
```

同时，还有一个没有后缀的 API，其具体实现取决于 `UNICODE` 宏是否定义：

```c
#ifdef UNICODE
   #define MessageBox MessageBoxW
#else
   #define MessageBox MessageBoxA
#endif
```

为了迎合这个没有后缀的 API，用同样的方式定义了一个字符类型 `TCHAR`：

```c
#ifdef UNICODE
    typedef wchar_t TCHAR;
#else
    typedef char TCHAR;
#endif
```

---

> 参考资料：  
> [Unicode - 维基百科，自由的百科全书](https://zh.wikipedia.org/zh-hans/Unicode)  
> [UTF-8 - 维基百科，自由的百科全书](https://zh.wikipedia.org/wiki/UTF-8)  
> [UTF-16 - 维基百科，自由的百科全书](https://zh.wikipedia.org/wiki/UTF-16)  
> [字节顺序标记 - 维基百科，自由的百科全书](https://zh.wikipedia.org/wiki/%E4%BD%8D%E5%85%83%E7%B5%84%E9%A0%86%E5%BA%8F%E8%A8%98%E8%99%9F)  
> [《流畅的 Python》](https://www.amazon.cn/dp/B072HMKKPG)  
> [unicode - Difference between MBCS and UTF-8 on Windows - Stack Overflow](https://stackoverflow.com/questions/3298569/difference-between-mbcs-and-utf-8-on-windows)  
> [windows - Why isn't UTF-8 allowed as the "ANSI" code page? - Stack Overflow](https://stackoverflow.com/questions/3298569/difference-between-mbcs-and-utf-8-on-windows)  
