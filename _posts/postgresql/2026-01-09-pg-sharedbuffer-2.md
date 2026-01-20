---
title: PostgreSQL 数据读取到缓存
author: amwps290
date: 2026-01-09 00:10:00 +0800
categories: [数据库, 源码分析, PostgreSQL]
tags: [database, postgresql]
mermaid: true
---

在这篇文章中我们将来查看一下数据库如何将磁盘上的数据文件，读入到缓存中。

这里我们要先说一下调试的方法。在启动一个 SQL 终端以后，例如 psql 我们需要使用 `select pg_backend_pid()；` 来获取一下我们连接连接的后端进程的id, 然后我们使用 gdb attach 的方式来连接到这个进程。同时如果调试中有 `SIGUSR1` 和 `SIGUSR2` 信号我们可以在 gdb 中进行屏蔽。

当我在数据库中想要查看某个表中的数据的时候，数据库就会将这个表的数据加载到我们上一节中讲述的共享缓存中。

```c
//src/backend/storage/buffer/bufmgr.c
Buffer
ReadBuffer(Relation reln, BlockNumber blockNum)
{
	return ReadBufferExtended(reln, MAIN_FORKNUM, blockNum, RBM_NORMAL, NULL);
}
```

在 `bufmgr.c` 文件中我们可以看到一个 `ReadBuffer` 这个函数就是用来将磁盘上的数据文件读取到内存中，这个函数直接调用来一个 `ReadBufferExtended`, 我们可以看到这个函数有两个参数，一个是 `Relation` 这个就是要将哪个表的数据或者索引读入到内存中，第二个就是块号，就是要将属于该关系的第几块数据读入到内存中。接下来我们看一下 `ReadBufferExtended` 函数。

```c
////src/backend/storage/buffer/bufmgr.c
Buffer
ReadBufferExtended(Relation reln, ForkNumber forkNum, BlockNumber blockNum,
				   ReadBufferMode mode, BufferAccessStrategy strategy)
{
	bool		hit;
	Buffer		buf;

	/*
	 * Reject attempts to read non-local temporary relations; we would be
	 * likely to get wrong data since we have no visibility into the owning
	 * session's local buffers.
	 */
	if (RELATION_IS_OTHER_TEMP(reln))
		ereport(ERROR,
				(errcode(ERRCODE_FEATURE_NOT_SUPPORTED),
				 errmsg("cannot access temporary tables of other sessions")));

	/*
	 * Read the buffer, and update pgstat counters to reflect a cache hit or
	 * miss.
	 */
	pgstat_count_buffer_read(reln);
	buf = ReadBuffer_common(RelationGetSmgr(reln), reln->rd_rel->relpersistence,
							forkNum, blockNum, mode, strategy, &hit);
	if (hit)
		pgstat_count_buffer_hit(reln);
	return buf;
}
```

在这个函数中我们可以看到主要分成了两个部分，一个是检查一下当前要加载数据的关系是不是另外一个会话的临时表，因为临时表是针对于一个会话的，
会话关闭，临时表就会被关闭。

接下来我们将调用 `ReadBuffer_common` 函数来读取数据。


```c
static Buffer
ReadBuffer_common(SMgrRelation smgr, char relpersistence, ForkNumber forkNum,
				  BlockNumber blockNum, ReadBufferMode mode,
				  BufferAccessStrategy strategy, bool *hit)
{
  // 变量申明
  
}
```
