---
title: Redis 数据结构源码分析
date: 2023-06-06 19:11
tags:
- redis
- IT
categories:
- redis
---
#redis

# 字符串
## Level 1 使用 `char []` 存储字符串
最简单的即使用 `char []` 来存储字符串，定义函数 `char* sdsnew(const char *init);` 
使用 `char[]` 最大的问题是 `C` 语言自身不记录字符串长度，可能会造成越界、缓冲区溢出问题，需要开发人员格外注意。
## Level 2 使用结构体定义存储结构
新增一个 `length` 属性来记录字符串长度，如下：
```c
struct stringStr {
    int length;
    char buf[];
};

struct stringStr* stringNew(const char *init);
```

这里就需要考虑一个问题了：`length` 属性的类型是什么？是 `short` 、`int`、`long` ？
如果是 `int` ，万一 `buf` 数组很长怎么办？
如果是 `long`，如果 `buf‵只存一个字节，那岂不是很浪费？
## Level 3 使用 SDS 
在 Redis 实现中，没有使用 C 原生的 `char []` 来表示字符串，而是自己实现了一个 SDS 的结构来存储字符串。该结构模式如下：
```c
// Define SDS struct
struct __attribute__ ((__packed__)) sdshdr5 {
    unsigned char flags;
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len;
    uint8_t alloc;
    unsigned char flags;
    char buf[];
};

struct __attribute__ ((__packed__)) sdshdr16 {
    uint16_t len;
    uint16_t alloc;
    unsigned char flags;
    char buf[];
};

struct __attribute__ ((__packed__)) sdshdr32 {
    uint32_t len;
    uint32_t alloc;
    unsigned char flags;
    char buf[];
};

struct __attribute__ ((__packed__)) sdshdr64 {
    uint64_t len;
    uint64_t alloc;
    unsigned char flags;
    char buf[];
};
```
解释一下结构体属性：
* `len` 表示字符串长度，可以 `O(1)` 获取
* `alloc` 除去头（`len` 、`alloc`、`flags` 占用的总大小）的长度和 `\0` 的内存分配大小
* `flags` 表示结构体类型
* `buf[]` 为实际存储字符串的字节数组，兼容 `C` 字符串

`flags` 定义了结构体的类型，且只使用了 3 位：
```c
// Define redis SDS type
#define SDS_TYPE_5 0
#define SDS_TYPE_8 1
#define SDS_TYPE_16 2
#define SDS_TYPE_32 3
#define SDS_TYPE_64 4
#define SDS_TYPE_MASK 7
```
如此，就可以通过字符串的长度，动态选择存储结构。
这里需要考虑一个问题，虽然可以动态创建了存储结构，那返回值要怎么表示？
返回可以通过返回一个带有 `flags` 的结构体，调用方根据 `flags` 将指针转成对应的类型。

`Redis` 并没有使用这种方式，`Redis` 返回的代表 `SDS` 的指针是这样的：
```c
typedef char * sds;
sds sdsnew(const char *init);
```

`__attribute__ ((__packed__))` 是一种特殊编译选项，用于告诉编译器该结构体存储不进行字节对齐，即内容多大就占用多大，如此，`Redis` 返回的 `sds` 指针实际上指向的是结构体 `char buf[]` 数组的第一个元素，则只需要往前挪一位即可获取到数据类型，通过类型的大小进行计算可以获取到整个结构体的所有数据，可以说是非常巧妙了！
相关代码如下：
```c
sds s;
char type = sdsReqType(initlen);
fp = ((unsigned char*)s)-1;
*fp = type;
```
