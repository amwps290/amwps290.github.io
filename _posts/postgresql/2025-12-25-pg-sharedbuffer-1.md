---
title: PostgreSQL 共享缓冲区初始化分析（一）
author: amwps290
date: 2025-12-25 00:10:00 +0800
categories: [数据库,源码分析,PostgreSQL]
tags: [database,postgresql]
mermaid: true
---


PostgreSQL 中有一个配置参数 `shared_buffers`，它控制着数据库将使用多少内存来缓存从磁盘读取的数据块。本文我们将深入分析共享缓冲区的初始化过程。

```c
//src/backend/storage/ipc/ipci.c
void
CreateSharedMemoryAndSemaphores(void)
```

首先，我们来看 `CreateSharedMemoryAndSemaphores` 函数。根据注释，该函数负责初始化共享内存和信号量。本文我们重点关注其中与共享内存相关的代码。

在函数的开头，有一段计算 `size` 的代码，用于确定需要申请的内存总量。这里会累加各个模块所需的内存，包括一个固定大小（约 100000 字节）的杂项结构体。其中有一行代码特别关键，它对应着配置文件中 `shared_buffers` 参数所指定的大小：

```c
size = add_size(size, BufferShmemSize());
```
接下来我们深入该函数：

```c
//src/backend/storage/buffer/buf_init.c
Size
BufferShmemSize(void)
{
	Size		size = 0;

	/* size of buffer descriptors */
	size = add_size(size, mul_size(NBuffers, sizeof(BufferDescPadded)));
	/* to allow aligning buffer descriptors */
	size = add_size(size, PG_CACHE_LINE_SIZE);

	/* size of data pages */
	size = add_size(size, mul_size(NBuffers, BLCKSZ));

	/* size of stuff controlled by freelist.c */
	size = add_size(size, StrategyShmemSize());

	/* size of I/O condition variables */
	size = add_size(size, mul_size(NBuffers,
								   sizeof(ConditionVariableMinimallyPadded)));
	/* to allow aligning the above */
	size = add_size(size, PG_CACHE_LINE_SIZE);

	/* size of checkpoint sort array in bufmgr.c */
	size = add_size(size, mul_size(NBuffers, sizeof(CkptSortItem)));

	return size;
}
```
`BufferShmemSize` 函数计算的是共享缓冲区所需的总内存大小，包括：
- 缓冲区描述符（`BufferDescPadded`）
- 实际存放数据页的内存（每个页大小为 `BLCKSZ`）
- 由 `freelist.c` 管理的结构（`StrategyShmemSize`）
- I/O 条件变量（`ConditionVariableMinimallyPadded`）
- 检查点排序数组（`CkptSortItem`）

此外，为了对齐缓存行，还额外添加了 `PG_CACHE_LINE_SIZE` 的填充。

回到 `CreateSharedMemoryAndSemaphores` 函数，接下来会执行：

```c
seghdr = PGSharedMemoryCreate(size, &shim);

InitShmemAccess(seghdr);
```

`PGSharedMemoryCreate` 负责向操作系统申请共享内存。该函数内部会根据配置选择内存申请方式：默认使用 `mmap` 申请一大块内存，同时通过 `shmget` 申请一个较小的共享内存段，用于后续的读写控制。除了默认的 `mmap` 方式，还可以通过 GUC 参数配置其他申请方式（如 `posix`、`sysv`）。函数最终返回一个 `PGShmemHeader` 指针，该指针指向已申请的内存段头部。

接下来，主流程会调用 `InitShmemAccess` 函数：

```c
//src/backend/storage/ipc/shmem.c
void
InitShmemAccess(void *seghdr)
{
	PGShmemHeader *shmhdr = (PGShmemHeader *) seghdr;

	ShmemSegHdr = shmhdr;
	ShmemBase = (void *) shmhdr;
	ShmemEnd = (char *) ShmemBase + shmhdr->totalsize;
}
```
`InitShmemAccess` 将刚刚申请的内存段通过三个全局变量进行管理：
- `ShmemSegHdr`：指向共享内存头部结构
- `ShmemBase`：指向内存起始地址
- `ShmemEnd`：指向内存结束地址

此后，所有对共享内存的操作都将基于这三个变量进行。由于它们是全局变量，必须通过锁机制来避免数据竞争。

```c
/*
    * Set up shared memory allocation mechanism
    */
if (!IsUnderPostmaster)
    InitShmemAllocation();

/*
    * Now initialize LWLocks, which do shared memory allocation and are
    * needed for InitShmemIndex.
    */
CreateLWLocks();
```
这段代码首先初始化共享内存分配机制（`InitShmemAllocation`），然后创建轻量级锁（`CreateLWLocks`）。轻量级锁用于保护共享内存的并发访问，同时也会为后续的 `InitShmemIndex` 做准备。

```c
InitShmemIndex();
```
`InitShmemIndex` 初始化一个哈希表，用于快速查找共享内存中的各个结构。在该函数内部，会调用 `ShmemInitStruct` 从共享内存中划分出一块内存用于存储该哈希表。后续许多其他模块也会通过 `ShmemInitStruct` 在共享内存中初始化自己的数据结构。

接下来是 `InitBufferPool` 函数：
```c
InitBufferPool();
```
该函数负责初始化缓冲区池，主要包括：
- `BufferDescriptors`：所有缓冲区描述符的数组，每个描述符对应一个共享内存块（buffer block）
- `BufferBlocks`：实际存放数据页的共享内存块数组
- `BufferIOCVArray`：I/O 条件变量数组，用于缓冲区 I/O 的同步
- 检查点相关的排序数组（本文暂不展开）

其余代码则是对这些结构进行进一步的初始化。

至此，PostgreSQL 的共享内存已经初始化完毕。接下来，数据库便可以将磁盘上的数据块缓存到这片内存中，从而加速后续的读写操作。

需要注意的是，共享内存的初始化由 postmaster 进程完成，而实际的读写操作则主要由后端进程（即连接到数据库的客户端进程）执行。

