---
title: 二、Redis 数据类型
date: 2023-05-25 08:26
tags:
- redis
- IT
categories:
- redis
---
#redis

`Redis` 是 K-V 数据库，数据类型指的是 `Value` 的数据类型。`Redis` 的 `Key` 可以使用任意二进制序列作为 `Key` （一般使用字符串），不建议太长，`Key` 太长会增加查找时间成本，太短则不可读，官方建议严格遵循一种规范来约束 `Key` 的命名，比如 `object-type:id` ，多词可以使用 `.` 或者 `-` 分割，例如 `comment:4321:reply.to` 或者 `comment:4321:reply-to`。`Redis Key` 最大允许 512 MB。

`Redis` 数据类型主要有以下几类：
* `Strings`
* `Lists`
* `Sets`
* `Hashes`
* `Sorted Sets`
* `Streams`
* `Geospatial`
* `HyperLogLog`
* `Bitmaps`
* `Bitfields`

