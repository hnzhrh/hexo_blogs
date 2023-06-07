---
title: Redis 数据结构源码分析
date: 2023-06-06 19:11
tags:
- redis
- IT
categories:
- redis
---
#redis #draft

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

## 代码实现
仿照 Redis 的实现，实现一个简单版本的 SDS，代码如下：
```c
//
// Created by erpang on 6/6/23.
//
#include <string.h>
#include <malloc.h>
#include <assert.h>
#include "../util/log.h"
#include "sds.h"
#include "../memory/sdsalloc.h"

const char *SDS_NOINIT = "SDS_NOINIT";

// 根据字符串长度，获取SDS结构类型
static inline char sdsReqType(size_t string_size) {
    if (string_size < 1<<5)
        return SDS_TYPE_5;
    if (string_size < 1<<8)
        return SDS_TYPE_8;
    if (string_size < 1<<16)
        return SDS_TYPE_16;
#if (LONG_MAX == LLONG_MAX)
    if (string_size < 1ll<<32)
        return SDS_TYPE_32;
    return SDS_TYPE_64;
#else
    return SDS_TYPE_32;
#endif
}

// 获取SDS最大可支持内存
static inline size_t sdsTypeMaxSize(char type) {
    if (type == SDS_TYPE_5)
        return (1<<5) - 1;
    if (type == SDS_TYPE_8)
        return (1<<8) - 1;
    if (type == SDS_TYPE_16)
        return (1<<16) - 1;
#if (LONG_MAX == LLONG_MAX)
    if (type == SDS_TYPE_32)
        return (1ll<<32) - 1;
#endif
    return -1; /* this is equivalent to the max SDS_TYPE_64 or SDS_TYPE_32 */
}


// 获取SDS类型对应的结构体占用大小
static inline int sdsHdrSize(char type) {
    switch(type&SDS_TYPE_MASK) {
        case SDS_TYPE_5:
            return sizeof(struct sdshdr5);
        case SDS_TYPE_8:
            return sizeof(struct sdshdr8);
        case SDS_TYPE_16:
            return sizeof(struct sdshdr16);
        case SDS_TYPE_32:
            return sizeof(struct sdshdr32);
        case SDS_TYPE_64:
            return sizeof(struct sdshdr64);
    }
    return 0;
}

sds _sdsnewlen(const void *init, size_t initlen, int trymalloc) {
    // 根据字符串长度判断SDS类型
    char type = sdsReqType(initlen);
    logger_debug("initlen:%d, type:%d\n", initlen, type);
    // 这里应该是为了性能考虑，空字符串通常用作拼接，直接用SDS_TYPE_8更可靠
    if (type == SDS_TYPE_5 && initlen == 0) type = SDS_TYPE_8;
    // 计算SDS头部大小
    int hdrlen = sdsHdrSize(type);

    // 这里是为了防止字符串没有溢出，但如果加上SDS头和终止符可能溢出的情况
    assert(initlen + hdrlen + 1 > initlen); /* Catch size_t overflow */

    logger_debug("hdrlen:%d, type:%d\n", hdrlen, type);
    void *sh;
    size_t usable;
    // 分配SDS内存大小为hdrlen + initlen + 1,即SDS头+char数组+终止符\0,分配的内存大小为usable的值
    sh = s_malloc_usable(hdrlen + initlen + 1, &usable);
    logger_debug("point sh[%p]\n", sh);

    if (sh == NULL) return NULL;
    logger_debug("malloc size:%d\n", usable);

    // TODO 此段代码意义未明
    if (init == SDS_NOINIT)
        init == NULL;
    else if (!init) {
        memset(sh, 0, hdrlen + initlen + 1);
    }

    // 定义SDS
    sds s;

    // SDS 指针指向char[] 第一个元素
    s = ((char*)sh) + hdrlen;
    logger_debug("point s[%p] char:%c\n", s, *s);

    // SDS头 flags指针，用来赋值SDS类型
    unsigned char *fp;
    fp = ((unsigned char*)s) - 1;
    logger_debug("point fp[%p] fp:%d\n", fp, *fp);


    // 记录已用内存
    usable = usable - hdrlen - 1;

    // 如果超出类型支持最大值，则置为最大值，这也就是为什么Redis String 最大支持512MB的原因
    if (usable > sdsTypeMaxSize(type))
        usable = sdsTypeMaxSize(type);

    // 根据不同SDS类型处理SDS结构
    switch(type) {
        case SDS_TYPE_5: {
            // 高5位存储字符串长度，低3为存储SDS类型，为了节约空间
            // 在用sds类型获取到flags后，flags&SDS_TYPE_BIT即可获取到SDS类型，右移3位后为字符串长度
            *fp = type | (initlen << SDS_TYPE_BITS);
            logger_debug("point fp[%p] fp:%d\n", fp, *fp);
            break;
        }
        case SDS_TYPE_8: {
            SDS_HDR_VAR(8,s);
            logger_debug("point sh[%p]，s[%p]\n", sh, s);
            sh->len = initlen;
            sh->alloc = usable;
            *fp = type;
            break;
        }
        case SDS_TYPE_16: {
            SDS_HDR_VAR(16,s);
            sh->len = initlen;
            sh->alloc = usable;
            *fp = type;
            break;
        }
        case SDS_TYPE_32: {
            SDS_HDR_VAR(32,s);
            sh->len = initlen;
            sh->alloc = usable;
            *fp = type;
            break;
        }
        case SDS_TYPE_64: {
            SDS_HDR_VAR(64,s);
            sh->len = initlen;
            sh->alloc = usable;
            *fp = type;
            break;
        }
    }
    // 拷贝字符数组到sds
    if (initlen && init)
        memcpy(s, init, initlen);
    // 添加终止符，兼容C字符串
    s[initlen] = '\0';
    return s;
}

sds sdsnew(const char *init) {
    size_t initlen = (init == NULL) ? 0 : strlen(init);
    return sdsnewlen(init, initlen);
}

sds sdsnewlen(const void *init, size_t initlen) {
    return _sdsnewlen(init, initlen, 0);
}
```

测试代码：
```c
#include <stdio.h>
#include "data-structure/sds.h"

sds testCreateSds(char *s) {
    return sdsnew(s);
}

int main() {
    sds str = testCreateSds("abcdefghijklasdfaasdfasdfasdfasdfasdfasdfasdfasdfasdfa");
    printf("sds:%s\n", str);
    return 0;
}

```

输出：
```sh
/home/erpang/c_projects/mini-redis/cmake-build-debug/mini_redis
[/home/erpang/c_projects/mini-redis/data-structure/sds.c,66] initlen:54, type:1
[/home/erpang/c_projects/mini-redis/data-structure/sds.c,75] hdrlen:3, type:1
[/home/erpang/c_projects/mini-redis/data-structure/sds.c,80] point sh[0x55693bd096b0]
[/home/erpang/c_projects/mini-redis/data-structure/sds.c,83] malloc size:72
[/home/erpang/c_projects/mini-redis/data-structure/sds.c,97] point s[0x55693bd096b3] char:
[/home/erpang/c_projects/mini-redis/data-structure/sds.c,102] point fp[0x55693bd096b2] fp:0
[/home/erpang/c_projects/mini-redis/data-structure/sds.c,123] point sh[0x55693bd096b0]，s[0x55693bd096b3]
sds:abcdefghijklasdfaasdfasdfasdfasdfasdfasdfasdfasdfasdfa
```