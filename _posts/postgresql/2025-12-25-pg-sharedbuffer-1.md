---
title: PostgreSQL shared buffer 分析-1
author: amwps290
date: 2025-12-25 00:10:00 +0800
categories: [数据库,源码分析,PostgreSQL]
tags: [database,postgresql]
mermaid: true
---


在 PostgreSQL 中有一个配置参数 `shared_buffer` 这个参数控制这数据库将使用多少内存来缓存从磁盘读取的数据块。今天我们先来看一下共享缓存的初始化。

<% gist 60e6af5a9adfe6e3103acbc853d5dedc CreateSharedMemoryAndSemaphores %>