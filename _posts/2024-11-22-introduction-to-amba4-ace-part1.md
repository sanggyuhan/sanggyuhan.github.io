---
title: Introduction to AMBA4 ACE - Part 1
description: >-
  The necessity and basic concepts of AMBA4 ACE
author:
date: 2024-11-22 23:24:00 +0900
categories: [Architecture, AMBA]
tags: [AMBA, ACE]
pin: false
media_subpath: '/posts/20241122'
---

## Why Is Cache Coherency Important?
Why is cache coherency important?
- In order to maintain or increase performance while maintaining energy consumption, the use of multi-processing such as big.LITTLE has become important.
- Multi-processing is an architecture that combines a processor cluster that supports high-performance with multi-issue and out-of-order and a high-efficiency with in-order processor cluster.
- Since a high-performance cluster and a high-efficiency cluster are architecturally compatible, there is no difference between the two processor clusters from a software perspective. In other words, one software can be executed identically on any processor cluster, so shared data transfer between processor clusters is widespread.
- Therefore, fast and power-efficient switching between the big and little cores is enabled by cache coherency between the processor clusters. Cache coherency enables cached data to be transferred between the clusters without being written back to external memory, saving time and energy.
- Meanwhile, as GPUs and specialized accelerators were added to SoCs, there were cases where shared data had to be shared between the elements.
- With the proliferation of caches, the challenge is to provide consistent views of shared data and to minimize the external memory bandwidth requirement, which is increasingly the main system performance bottleneck.
- As cache coherency becomes more important, it has stimulated the demand for a flexible cache coherency protocol that can be used by not only homogeneous CPUs but also heterogeneous CPUs, GPUs, and hardware accelerators.


## The Coherency Challenge
There are potential coherency challenges that can occur when using one or more caches.
- When memory is updated after a cached master has a copy of it. In this case, the data in the cache becomes out-of-date. This can happen with any type of cache such as write-through, write-back.
- When a cached master updates its own cache. The data in memory becomes out-of-date. When another master accesses the memory at that location, it reads out-of-date, stale, data. This can happen if the master has a write-back cache.


## Approaches to Solve Coherency Challenge
- Solution 1 - Software-based coherency approach
  - When cached copy is stale, invalidate the cached copy and reread the data from main memory when the data is needed.
  - When main memory is stale, clean the write-back cache that has dirty data (forcing write-back) and invalidate the cached copy of other masters.
  - Remaining problem: The cost of software development is too high. For example, if the processor directly performs the cache invalidating and cleaning operations, it has a negative impact on performance and greatly increases software complexity. Therefore, it is better to apply other methods that can reduce software complexity and timescales.
- Solution 2 - Hardware-based coherency approach - Snooping cache coherency protocols
  - All masters listen in on all shared-data transactions that occurred from other masters. Masters have address inputs for snooping as well as address outputs. When a master detects a read transaction, it provides the data if it has up-to-date data. And when detecting a write transaction, it invalidates its local copy. (Does it have to be invalidated or can it be changed to SharedClean state? A. To be in SharedClean state, the local cache must be kept up to date with the latest data. Therefore, it seems right to invalidate it.)
  - If there are N masters in the system, a total of N*(N-1) snoop channels should be added. This means that it can be useful only in systems with a small number of processors (2~8). If the system uses more masters, it can filter unnecessary coherency traffic by using a ‘snoop filter’ that acts as a partial directory. A snoop-based system with a fully populated snoop filter is similar to a directory-based system.
- Solution 3 - hardware-based coherency approach - Directory-based cache coherency protocols
  - In a directory-based system, there is a single ‘directory’ that lists which master’s cache stores each cached line in the system. When a master initiates a transaction, the master checks the directory to see which masters have cached the area corresponding to the transaction address. It then generates cache coherency traffic only for the masters that have a cached copy. Directory-based systems scale better than purely snooping systems that broadcast coherency traffic to all masters.
  - In the ideal case where shared data is used by only two masters, only 2N transactions need to occur. In the worst case where shared data is shared by all masters, N*(N-1+1) = N^2 (+1 due to directory lookup) transactions are required.
  - The downside of directory-based systems is that they require additional resources for the directory and additional latency for directory access.


## AMBA4 ACE Cache Coherency States
- ACE cache state corresponds to the MOESI model, but components that use other cache coherency state models such as MESI and MEI can also use ACE.
- For example, Cortex-A15 MPCore cannot be in Owned (SharedDirty) state because it uses MESI state internally.
- Cache state is allocated by cache line.
- When data is Shared, all cached copies must always have the latest data. However, only one copy can be in SharedDirty state, and the rest must be in SharedClean state. In other words, only the cache that has the SharedDirty copy is responsible for writing the copied data to memory.
- Snooping is when the interconnect issues a snooping transaction to another cached master to read the data and state of the cached master. Therefore, there is no way to inform other masters oßf the data to be updated when the initiating master is performing a cache line update. So, after invalidating the cache of another master, a write is performed.



| Cache State Model | ACE         | ACE Meaning           |
|-------------------|-------------|-----------------------|
| M (Modified)      | UniqueDirty | Not shared, dirty, must be written back to memory <br> (U) The cache line exists only in this cache. <br> (D) The cache line was modified relative to the memory (most up-to-date data), and the master must notify the memory of the changed cache line at some point. <br> A component (cached master) can store to a cache line in the UniqueDirty state without notifying other caches. |
| O (Owned)         | SharedDirty | Shared, dirty, must be written back to memory <br> (S) The cache line may have been shared with other caches. (may not necessarily be shared.) <br> (D) The cache line has been modified relative to memory, and the master must notify memory of the changed cache line at some point. <br> When updating the cache line, the other caches must be notified of the change. |
| E (Exclusive)     | UniqueClean | Not shared, clean <br> (U) The cache line exists only in this cache. <br> (C) It has not been modified from the value in main memory. <br> A component can store to a cache line without notifying other caches. |
| S (Shared)        | SharedClean | Shared, no need to write back, may be clean or dirty (when there is another cache in the SharedDirty state) <br> (S) The cache line may be shared with another cache. (This is not necessarily shared, since there is no need to notify this cache when a cache line in another cache is discarded.) <br> (C) The cache line contains the most up-to-date data, but it does not know whether it has been modified relative to memory. This component is not responsible for updating the value in memory. <br> When updating the cache line, the other caches must be notified of the change. (Because it is shared) |
| I (Invalid)       | Invalid     | Invalid <br> Cache line doest not exist in cache. |



## Sources
Ashley Stevens, Introduction to AMBA^®^ 4 ACE^TM^ and big.LITTLE^TM^ Processing Technology, 2011