---
title: 一、Redis初探
date: 2023-05-24 21:51
tags:
- redis
- IT
categories:
- redis
---
#redis
# 什么是 Redis？
> The open source, in-memory data store used by millions of developers as a database, cache, streaming engine, and message broker.
# Redis 的特性
## 丰富的数据类型
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
## 可编程
可以通过 `Lua` 脚本和 `Redis Functions` 编程。
## 可扩展模块
`Redis` 提供了 `module API`，可以通过 `C`、`C++`、`Rust` 进行定制化扩展。例如增加布隆过滤器模块，使 `Redis` 拥有布隆过滤器功能。
## 持久化
`Redis` 除了支持内存高速访问的同时，还支持将数据持久化，避免系统崩溃、重启导致的数据丢失。
## 集群
`Redis` 基于哈希分区机制支持海量数据的存储。
## 高可用
无论是单机还是集群部署，`Redis` 都提供了基于副本的故障转移能力。
# 参考资料
* [Redis](https://redis.io/)