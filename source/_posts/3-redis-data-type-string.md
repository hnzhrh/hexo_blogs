---
title: 三、Redis 数据类型 String 详解
date: 2023-05-25 22:41
tags:
- redis
- IT
categories:
- redis
---
#redis

# 简介
`String` 类型是 `Redis` 最简单的数据类型，也可以存储二进制数据，最大为 512 MB。在不同命令的作用下，`Value` 会转成对应的类型，比如：
```shell
set counter 100
incr counter
```
`INCR` 命令会将 `String` 转为 `Integer`。

# 命令
`String` 类型的操作命令可以分为以下几类：
* 单值读写
	* `SET`
	* `GET`
	* `GETDEL`
	* `GETSET`
* 多值读写
	* `MGET`
	* `MSET`
	* `MSETNX`
* 字符串函数操作
	* `APPEND`
	* `GETRANGE`
	* `LCS`
	* `SETRANGE`
	* `STRLEN`
	* `SUBSTR`
* 数字运算
	* `DECR`
	* `DECRBY`
	* `INCR`
	* `INCRBY`
	* `INCRBYFLOAT`
* 存活时间相关操作
	* `GETEX`
以上命令中，最重要的是 `SET` 命令，通过多个参数控制行为：
```shell
SET key value [NX | XX] [GET] [EX seconds | PX milliseconds |
  EXAT unix-time-seconds | PXAT unix-time-milliseconds | KEEPTTL]
```
* `EX` ，设置秒级过期时间
* `PX`，设置毫秒级过期时间
* `EXAT`，设置秒级过期 Unix 时间戳
* `PXAT`，设置毫秒级过期 Unix 时间戳
* `NX`，仅在不存在 `Key` 时设置 `Value`
* `XX`，仅在存在 `Key` 时设置 `Value`
* `GET`，返回旧值
* `KEEPTTL`，保持当前 `Key` 的存活时间
`KEEPTTL` 的作用是如果当前 `Key` 有存活时间，则在更新值后保持原有的存活时间不变。Demo：
```shell
> SET user:name erpang EX 60
"OK"

> TTL user:name
(integer) 56

> SET user:name hnzhrh
"OK"

> TTL user:name
(integer) -1

> SET user:name erpang EX 60
"OK"

> TTL user:name
(integer) 55

> SET user:name hnzhrh KEEPTTL
"OK"

> TTL user:name
(integer) 37

```
# 应用场景 (TODO，生产环境的实现，而不是 Demo)
可以使用 `String` 类型做一些数据的缓存，利用 `Redis` 的过期机制、原子性和高并发特性来实现业务场景。
## 缓存认证 Token
例如缓存账号 ID 为3254234的 `JWT` 新信息：
```shell
SET user:token:3254234 eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyIjoiZXJwYW5nIn0.kqZa9MkhpRYtZJA9NSkHn-ND8e1g-NFxIJZUG00juSw ex 600
```

## 简单分布式锁
使用 `NX` 参数确保只有一个客户端可以成功 `SET` 数据，即获取到锁，同时为了防止客户端出现异常，一直持有锁导致死锁，加上过期时间。
```shell
SET task:1000 true EX 60 NX
DEL task:1000
```
PS：这个锁是有问题的，如果业务执行时间超过 60 s，则该锁会自动释放，需要在锁的客户端实现上加上机制来避免该问题，比如 Watch dog 机制。
## 计数器
可以使用 `INCR` 指令实现阅读量、访问量，比如博客 ID 为 121 的阅读量增加 1。
```shell
INCR blog:readcount:121
```
PS：如果是高并法，这种方式是不可取的，对 Redis 压力太大。


