---
title: Introduction to AMBA4 ACE - Signals and Transactions
description: >-
  Signals and transactions of AMBA4 ACE
author:
date: 2024-11-30 09:49:00 +0900
categories: [Architecture, AMBA]
tags: [AMBA, ACE]
pin: false
media_subpath: '/posts/20241130'
---

## ACE Additional Channels and Signals

ACE adds three channels for cache coherency to the five channels of AMBA AXI4.
- ACADDR : Coherent address channel. Input to master
- CRRESP : Coherent response channel. Output from master
- CDATA : Coherent data channel. Output from master

Signals for coherency was added to the AXI4 existing address and data channels as well.
- Read address channel
  - ARSNOOP[3:0]: Type of snoop transactions for shareable transactions
  - ARBAR[1:0]: Barrier signaling
  - ARDOMAIN[1:0] : Which master should be snooped for snoop transactions and which master must be considered for ordering of barrier transactions
- Read data channel
  - RRESP[3:0]: Read response bit for snoop response sent from snooped master to CRRESP
  - RACK
- Write address channel
  - AWSNOOP[3:0]
  - AWBAR[1:0]
  - AWDOMAIN[1:0]
- Write data channel
  - WACK



## ACE Transactions

ACE added transactions to AXI4. Below is a group of added transactions.
- Non-shared
    - It is the same as read, write non-coherent, non-snooped transaction used in AXI.
    - Non-sharedable (no other master allocates cache line), transaction for device area.
    - ReadNoSnoop
    - WriteNoSnoop
- Shareable, Non-cached
  - Transactions that access the Shareable area but do not require a cached copy.
  - Unlike Non-shared group transactions, they generate a snoop transaction on the interconnect.
  - ReadNoSnoop, WriteNoSnoop, ReadOnce, and WriteUnique are transactions that do not operate on the entire cache line in ACE.
  - ReadOnce: Transaction used when a master reads shared data but does not store a copy. Other masters that have a copy do not have to change their cache state. In other words, a Unique master that provided the read data can keep its state as Unique. For example, a display controller reads a cached frame buffer.
  - WriteUnique: Used when an uncached master writes data to the Shared area. It means that all other cached copies must be cleaned. The interconnect sends a CleanInvalid command to the snoop channel, and the cached master writes the dirty copy to memory (WriteBack or WriteClean) and then invalidates the dirty and clean copies. - WriteLineUnique: Same as WriteUnique, except that it operates on the entire cache line. Since it operates on the entire cache line, other dirty copies do not need to be written back; they only need to be invalidated.
- Shareable Read: A type of coherent transactions, used to perform load operations on the Shareable area. Holders of the current cache line can still have a copy.
  - ReadClean: Used when the master requesting the read wants to accept a clean copy of the cache line. For example, if the master has a write-through cache, it cannot have data in a dirty state, so ReadClean is used.
  - ReadShared: Used when the master requesting the read can accept cache line data in any state.
  - ReadNotSharedDirty: Used when the master requesting the read can accept everything except the SharedDirty state. For example, this can apply when the master uses the MESI model, not MOESI.
- Shareable Write: Transactions used to gain write access to a copy of the internal write-back cache, so the Shareable write group is actually performed on the read channel.
  - MakeUnique: Used to invalidate all copies of a cache line before a Shareable write. The initiating master can allocate the cache line and perform writes after all copies have been cleaned and invalidated.
  - ReadUnique: Same as MakeUnique, except that it reads data from memory or another cached master. It is used before a partial line write. MakeUnique does not need to read up-to-date data, because it is used before a write that overwrites the entire cache line.
  - CleanUnique: Used before a partial line write when the master already has a copy of the cache line. It is similar to ReadUnique, except that it does not need to read data. It only needs to be written back to memory by another dirty copy and made invalid. As described in ACE coherency state, the SharedDirty state can have only one master, and all cache lines in the Shared state have up-to-date data. Therefore, we can expect that the initiating master of CleanUnique is in SharedClean state (if it is Unique, there is no need to send CleanUnique, and if it is SharedDirty, other masters with shared copies will be in clean state) and has an up-to-date copy of the data. We can also use ReadUnique instead of CleanUnique, although there is unnecessary read data.
- Write-back: The following transactions can be performed without requiring any snoop transactions.
  - WriteBack: Used when writing the entire dirty line to memory. After WriteBack, the cache line is invalid. It is mainly used by the operation of evicting the dirty line and allocating a new line.
  - WriteClean: Used when the master wants to maintain a clean state copy of the cache line when updating the memory. When eviction is not necessarily required, the master can write data to the memory while expecting that the copied data will not be updated until the eviction point. WriteClean is used at this time. WriteBack and WriteClean are used to enable the external snoop filter to track the cache contents.
  - WriteEvict: Used to evict the cache line. The data is written to the lower level of the cache hierarchy, and does not necessarily have to be written to the main memory.
  - Evict: Does not write back the data. It simply notifies that the cache line has been evicted. It is only used as a way for the external snoop filter to track the contents in the cache. 
- Cache Maintenance
  - CleanShared: A transaction that broadcasts to write a dirty copy to memory. The cache may have a copy of the data in a clean state.
  - CleanInvalid: A broadcast that writes a dirty copy to memory, just like CleanShared, except that the cache line must be made invalid.
  - MakeInvalid: A broadcast that makes all copies of the cache line invalid. Even if it is dirty, it does not need to be written back. MakeInvalid can occur as a response to CMO (Cache Maintenance Operation) or WriteLineUnique interconnect.


## Barriers

In systems that support shared-memory communication and out-of-order execution, barriers need to be used to ensure the ordering of transactions.
- DMB (Data Memory Barrier): All masters can verify that transactions that occurred before the DMB have finished before transactions that occurred after the DMB. In other words, it ensures that load and store operations after the DMB are executed after the load and store operations before the DMB have completed. If it is necessary to maintain coherence outside the cluster, the barrier is broadcast to the ACE interface. The value of the AxDOMAIN signal can be used to specify a subset of masters that the barrier will affect.
- DSB (Data Synchronization Barrier): Unlike DMB, which allows transactions to continue through the pipelined interconnect, DSB stalls the processor until all transactions before the barrier have completed. For example, DSB can be used to ensure that DMA is kicked off to the peripheral register after all data has been written to the DMA command buffer in memory. - You can set the range of domains to broadcast DMB/DSB by setting the value of the AxDOMAIN signal. In the case of the inner domain, code and data are shared (=running the same OS instance), and the outer domain does not share code but only data.


## Distributed Virtual Memory (DVM)

A multi-cluster coherent CPU system that shares a single set of MMU page tables (in memory) must ensure TLB coherency. The TLB (Translation Look-aside Buffer) is a cache of MMU page tables (in memory). When a master updates a page table entry, the master must invalidate any TLBs that may contain stale copies of the page table. ACE supports Distributed Virtual Memory (DVM), which consists of broadcast invalidation messages. DVM messages are transmitted using ARSNOOP signaling on the read channel. DVM messages support the following functions: 
- TLB invalidation: SMMU (System MMU) can keep its entries up-to-date by referring to TLB invalidation messages. 
- Branch predictor, virtual or physical instruction cache invalidation (for when a processor has written code to memory): Branch predictor, instruction cache invalidation messages are used for interprocessor communication by ensuring that instruction-side invalidations occur on all processes (including different clusters) in the system. 
- Synchronization: Used to wait for all previous DVM commands to finish.


## Source
Ashley Stevens, Introduction to AMBA® 4 ACE™ and big.LITTLE™ Processing Technology, 2011