---
title: PostgreSQL 数据读取到共享缓冲区
author: amwps290
date: 2026-01-09 00:10:00 +0800
categories: [数据库, 源码分析, PostgreSQL]
tags: [database, postgresql]
mermaid: true
---

在这篇文章中，我们将深入探讨 PostgreSQL 如何将磁盘上的数据文件读入到共享缓冲区中。

## 调试方法（可选）

如果你希望跟踪代码执行过程，可以使用以下调试方法：启动一个 SQL 终端（如 psql），执行 `SELECT pg_backend_pid();` 获取当前后端进程的 PID，然后通过 gdb attach 到该进程。调试过程中可能会收到 `SIGUSR1` 和 `SIGUSR2` 信号，可以在 gdb 中屏蔽它们。

## 数据读取入口

当查询需要访问某个表的数据时，PostgreSQL 会将该表对应的数据块加载到共享缓冲区中。这一过程始于 `ReadBuffer` 函数（位于 `src/backend/storage/buffer/bufmgr.c`）：

```c
// src/backend/storage/buffer/bufmgr.c
Buffer
ReadBuffer(Relation reln, BlockNumber blockNum)
{
    return ReadBufferExtended(reln, MAIN_FORKNUM, blockNum, RBM_NORMAL, NULL);
}
```

`ReadBuffer` 接收两个参数：
- `Relation reln`：目标关系（表或索引）
- `BlockNumber blockNum`：需要读取的块号

该函数直接调用 `ReadBufferExtended`，后者提供了更丰富的控制参数（如分支号、读取模式、缓冲策略等）。

## ReadBufferExtended

```c
// src/backend/storage/buffer/bufmgr.c
Buffer
ReadBufferExtended(Relation reln, ForkNumber forkNum, BlockNumber blockNum,
                   ReadBufferMode mode, BufferAccessStrategy strategy)
{
    bool        hit;
    Buffer      buf;

    /*
     * 拒绝读取非本会话的临时关系——因为我们无法看到其他会话的本地缓冲区。
     */
    if (RELATION_IS_OTHER_TEMP(reln))
        ereport(ERROR,
                (errcode(ERRCODE_FEATURE_NOT_SUPPORTED),
                 errmsg("cannot access temporary tables of other sessions")));

    /*
     * 读取缓冲区，并更新统计计数器以反映缓存命中或未命中。
     */
    pgstat_count_buffer_read(reln);
    buf = ReadBuffer_common(RelationGetSmgr(reln), reln->rd_rel->relpersistence,
                            forkNum, blockNum, mode, strategy, &hit);
    if (hit)
        pgstat_count_buffer_hit(reln);
    return buf;
}
```

该函数主要完成两项检查：
1. **临时表访问检查**：确保不会跨会话访问临时表（临时表的数据位于会话私有的本地缓冲区中）。
2. **统计计数**：记录缓冲区读取次数，并在命中时更新命中次数。

随后调用 `ReadBuffer_common` 执行实际的读取操作。

## ReadBuffer_common

`ReadBuffer_common` 是读取逻辑的核心，其代码较长，我们分段分析。

### 函数签名与变量声明

```c
static Buffer
ReadBuffer_common(SMgrRelation smgr, char relpersistence, ForkNumber forkNum,
                  BlockNumber blockNum, ReadBufferMode mode,
                  BufferAccessStrategy strategy, bool *hit)
{
    BufferDesc *bufHdr;
    Block       bufBlock;
    bool        found;
    bool        isExtend;
    bool        isLocalBuf = SmgrIsTemp(smgr);

    *hit = false;

    /* 确保有足够空间记录 buffer pin */
    ResourceOwnerEnlargeBuffers(CurrentResourceOwner);

    isExtend = (blockNum == P_NEW);

    TRACE_POSTGRESQL_BUFFER_READ_START(forkNum, blockNum,
                                       smgr->smgr_rnode.node.spcNode,
                                       smgr->smgr_rnode.node.dbNode,
                                       smgr->smgr_rnode.node.relNode,
                                       smgr->smgr_rnode.backend,
                                       isExtend);

    /* 如果调用者请求 P_NEW，则替换为正确的块号 */
    if (isExtend)
    {
        blockNum = smgrnblocks(smgr, forkNum);
        /* 如果关系已达到最大可能长度，则报错 */
        if (blockNum == P_NEW)
            ereport(ERROR,
                    (errcode(ERRCODE_PROGRAM_LIMIT_EXCEEDED),
                     errmsg("cannot extend relation %s beyond %u blocks",
                            relpath(smgr->smgr_rnode, forkNum),
                            P_NEW)));
    }
```

**关键点**：
- `isLocalBuf`：判断是否为临时表（使用本地缓冲区）。
- `ResourceOwnerEnlargeBuffers`：确保当前资源所有者（ResourceOwner）有空间记录即将增加的 buffer pin。这是为了在事务结束或回滚时能够正确释放 pin，防止缓冲区被永久固定在内存中。
- `isExtend`：判断是否为扩展关系（即请求新块）。若块号为 `P_NEW`，则通过 `smgrnblocks` 获取当前文件长度，并将该块作为新块分配。

### 缓冲区查找与分配

```c
    if (isLocalBuf)
    {
        bufHdr = LocalBufferAlloc(smgr, forkNum, blockNum, &found);
        if (found)
            pgBufferUsage.local_blks_hit++;
        else if (isExtend)
            pgBufferUsage.local_blks_written++;
        else if (mode == RBM_NORMAL || mode == RBM_NORMAL_NO_LOG ||
                 mode == RBM_ZERO_ON_ERROR)
            pgBufferUsage.local_blks_read++;
    }
    else
    {
        /*
         * 查找缓冲区。如果请求的块不在内存中，则会设置 IO_IN_PROGRESS 标志。
         */
        bufHdr = BufferAlloc(smgr, relpersistence, forkNum, blockNum,
                             strategy, &found);
        if (found)
            pgBufferUsage.shared_blks_hit++;
        else if (isExtend)
            pgBufferUsage.shared_blks_written++;
        else if (mode == RBM_NORMAL || mode == RBM_NORMAL_NO_LOG ||
                 mode == RBM_ZERO_ON_ERROR)
            pgBufferUsage.shared_blks_read++;
    }

    /* 此时我们不持有任何锁。 */
```

根据缓冲区类型（本地或共享）分别调用 `LocalBufferAlloc` 或 `BufferAlloc`。这两个函数均尝试在缓冲区池中查找目标块；若找到（`found == true`），则直接返回对应的缓冲区描述符，并更新相应的统计计数器（命中、写入或读取）。

`BufferAlloc` 的主要职责有两个：
1. 检查请求的磁盘块是否已在内存中。
2. 若不在，则分配一个空闲缓冲区（可能涉及淘汰旧页面），并初始化其描述符 `bufHdr`。

### 缓存命中处理

```c
    /* 如果已经在缓冲区池中，则直接返回 */
    if (found)
    {
        if (!isExtend)
        {
            /* 仅需更新统计信息后退出 */
            *hit = true;
            VacuumPageHit++;

            if (VacuumCostActive)
                VacuumCostBalance += VacuumCostPageHit;

            TRACE_POSTGRESQL_BUFFER_READ_DONE(forkNum, blockNum,
                                              smgr->smgr_rnode.node.spcNode,
                                              smgr->smgr_rnode.node.dbNode,
                                              smgr->smgr_rnode.node.relNode,
                                              smgr->smgr_rnode.backend,
                                              isExtend,
                                              found);

            /*
             * 在 RBM_ZERO_AND_LOCK 模式下，调用者期望返回时页面已被锁定。
             */
            if (!isLocalBuf)
            {
                if (mode == RBM_ZERO_AND_LOCK)
                    LWLockAcquire(BufferDescriptorGetContentLock(bufHdr),
                                  LW_EXCLUSIVE);
                else if (mode == RBM_ZERO_AND_CLEANUP_LOCK)
                    LockBufferForCleanup(BufferDescriptorGetBuffer(bufHdr));
            }

            return BufferDescriptorGetBuffer(bufHdr);
        }
```

若缓冲区已存在且不是扩展操作，则：
- 标记命中（`*hit = true`）。
- 更新 Vacuum 相关统计（用于 autovacuum 成本计算）。
- 根据读取模式决定是否加锁（如 `RBM_ZERO_AND_LOCK` 要求独占锁）。
- 最后返回缓冲区句柄。

### 扩展时的特殊情况

若在扩展关系时发现一个已存在且标记为 `BM_VALID` 的缓冲区，说明之前可能曾尝试读取超出 EOF 的块（例如由于内核 bug 或 `zero_damaged_pages` 设置）。此时会检查页面是否真的为全新（`PageIsNew`），若不是则报错。

```c
        /*
         * 我们仅在此角落情况下执行：试图扩展关系，但找到了一个已标记 BM_VALID 的缓冲区。
         * 这可能是因为 mdread 不会抱怨超出 EOF 的读取（当 zero_damaged_pages 为 ON），
         * 导致之前读取超出 EOF 的尝试留下了一个“有效”的零填充缓冲区。
         * 不幸的是，在某些有 bug 的 Linux 内核中也可能出现这种情况，
         * 其 lseek(SEEK_END) 的结果未能反映最近的写入。此时，已存在的缓冲区可能包含有效数据，
         * 我们不希望覆盖它。因为合法情况应始终留下零填充的缓冲区，
         * 所以若非 PageIsNew 则报错。
         */
        bufBlock = isLocalBuf ? LocalBufHdrGetBlock(bufHdr) : BufHdrGetBlock(bufHdr);
        if (!PageIsNew((Page) bufBlock))
            ereport(ERROR,
                    (errmsg("unexpected data beyond EOF in block %u of relation %s",
                            blockNum, relpath(smgr->smgr_rnode, forkNum)),
                     errhint("This has been seen to occur with buggy kernels; consider updating your system.")));

        /* 必须执行 smgrextend，否则内核不会保留该页面，下一次 P_NEW 可能会返回同一页面。
         * 清除 BM_VALID 位，执行 BufferAlloc 未做的 StartBufferIO 调用，然后继续。
         */
        if (isLocalBuf)
        {
            /* 仅需调整标志位 */
            uint32      buf_state = pg_atomic_read_u32(&bufHdr->state);

            Assert(buf_state & BM_VALID);
            buf_state &= ~BM_VALID;
            pg_atomic_unlocked_write_u32(&bufHdr->state, buf_state);
        }
        else
        {
            /* 循环处理极小的可能性：有人在我们清除 BM_VALID 和 StartBufferIO 检查之间重新设置了它 */
            do
            {
                uint32      buf_state = LockBufHdr(bufHdr);

                Assert(buf_state & BM_VALID);
                buf_state &= ~BM_VALID;
                UnlockBufHdr(bufHdr, buf_state);
            } while (!StartBufferIO(bufHdr, true));
        }
    }
```

### 缓存未命中：读取或扩展

若未找到缓冲区（`found == false`），则说明需要从磁盘读取数据（或扩展文件）。

```c
    /*
     * 走到这里意味着我们已经为页面分配了缓冲区，但其内容尚未有效。
     * 如果是共享缓冲区，则其 IO_IN_PROGRESS 标志已被设置。
     */
    Assert(!(pg_atomic_read_u32(&bufHdr->state) & BM_VALID));   /* 无需自旋锁 */

    bufBlock = isLocalBuf ? LocalBufHdrGetBlock(bufHdr) : BufHdrGetBlock(bufHdr);

    if (isExtend)
    {
        /* 新缓冲区用零填充 */
        MemSet((char *) bufBlock, 0, BLCKSZ);
        /* 全零页面不计算校验和 */
        smgrextend(smgr, forkNum, blockNum, (char *) bufBlock, false);

        /*
         * 注意：我们*没有*在此处调用 ScheduleBufferTagForWriteback；
         * 虽然我们本质上执行了一次写入。至少在 Linux 上，这样做会破坏“延迟分配”机制，
         * 导致文件碎片增加。
         */
    }
    else
    {
        /*
         * 读入页面，除非调用者打算覆盖它并只希望我们分配一个缓冲区。
         */
        if (mode == RBM_ZERO_AND_LOCK || mode == RBM_ZERO_AND_CLEANUP_LOCK)
            MemSet((char *) bufBlock, 0, BLCKSZ);
        else
        {
            instr_time  io_start,
                        io_time;

            if (track_io_timing)
                INSTR_TIME_SET_CURRENT(io_start);

            smgrread(smgr, forkNum, blockNum, (char *) bufBlock);

            if (track_io_timing)
            {
                INSTR_TIME_SET_CURRENT(io_time);
                INSTR_TIME_SUBTRACT(io_time, io_start);
                pgstat_count_buffer_read_time(INSTR_TIME_GET_MICROSEC(io_time));
                INSTR_TIME_ADD(pgBufferUsage.blk_read_time, io_time);
            }

            /* 检查数据是否损坏 */
            if (!PageIsVerifiedExtended((Page) bufBlock, blockNum,
                                        PIV_LOG_WARNING | PIV_REPORT_STAT))
            {
                if (mode == RBM_ZERO_ON_ERROR || zero_damaged_pages)
                {
                    ereport(WARNING,
                            (errcode(ERRCODE_DATA_CORRUPTED),
                             errmsg("invalid page in block %u of relation %s; zeroing out page",
                                    blockNum,
                                    relpath(smgr->smgr_rnode, forkNum))));
                    MemSet((char *) bufBlock, 0, BLCKSZ);
                }
                else
                    ereport(ERROR,
                            (errcode(ERRCODE_DATA_CORRUPTED),
                             errmsg("invalid page in block %u of relation %s",
                                    blockNum,
                                    relpath(smgr->smgr_rnode, forkNum))));
            }
        }
    }
```

- **扩展**：将缓冲区清零并调用 `smgrextend` 将空页面写入文件。
- **读取**：调用 `smgrread` 从磁盘读取数据到 `bufBlock`，同时记录 I/O 耗时（若开启 `track_io_timing`）。读取后使用 `PageIsVerifiedExtended` 验证页面完整性；若损坏，根据配置决定是报错还是清零页面。

### 标记缓冲区有效并返回

```c
    /*
     * 在 RBM_ZERO_AND_LOCK 模式下，在标记页面为有效之前获取缓冲区内容锁，
     * 以确保其他后端在调用者初始化之前不会看到零填充的页面。
     */
    if ((mode == RBM_ZERO_AND_LOCK || mode == RBM_ZERO_AND_CLEANUP_LOCK) &&
        !isLocalBuf)
    {
        LWLockAcquire(BufferDescriptorGetContentLock(bufHdr), LW_EXCLUSIVE);
    }

    if (isLocalBuf)
    {
        /* 仅需调整标志位 */
        uint32      buf_state = pg_atomic_read_u32(&bufHdr->state);

        buf_state |= BM_VALID;
        pg_atomic_unlocked_write_u32(&bufHdr->state, buf_state);
    }
    else
    {
        /* 设置 BM_VALID，终止 I/O，并唤醒任何等待者 */
        TerminateBufferIO(bufHdr, false, BM_VALID);
    }

    VacuumPageMiss++;
    if (VacuumCostActive)
        VacuumCostBalance += VacuumCostPageMiss;

    TRACE_POSTGRESQL_BUFFER_READ_DONE(forkNum, blockNum,
                                      smgr->smgr_rnode.node.spcNode,
                                      smgr->smgr_rnode.node.dbNode,
                                      smgr->smgr_rnode.node.relNode,
                                      smgr->smgr_rnode.backend,
                                      isExtend,
                                      found);

    return BufferDescriptorGetBuffer(bufHdr);
}
```

最后，根据缓冲区类型设置 `BM_VALID` 标志（本地缓冲区直接修改状态，共享缓冲区通过 `TerminateBufferIO` 完成），更新未命中统计，并返回缓冲区句柄。

## 总结

`ReadBuffer` 系列函数是 PostgreSQL 缓冲区管理的核心，其通过多层封装实现了：
1. **缓存查找**：优先从共享缓冲区或本地缓冲区中获取页面，避免不必要的磁盘 I/O。
2. **缓存分配**：若未命中，则分配一个缓冲区（可能触发页面淘汰）。
3. **磁盘 I/O**：通过 `smgrread` 或 `smgrextend` 读取或扩展数据。
4. **页面验证**：确保读取的数据完整有效。
5. **并发控制**：根据读取模式加锁，保证多后端访问的一致性。

理解这一流程有助于我们深入掌握 PostgreSQL 的存储层工作原理。
