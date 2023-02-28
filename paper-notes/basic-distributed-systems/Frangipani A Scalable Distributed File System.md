### Motivations

To achieve an ideal distributed file system, the system should be coherent, scalable, highly-available, highly-automated and simple. Especially, the aim of simplicity is always lost in current mass storage system, which heavily depends on manual operations, even some mechanisms indeed decrease some loads. 

A lot of distributed file system have tried to be ideal, but they fail to attain a good balance among those aims. Here, this paper proposes **Frangipani**, a distributed file system with a very simple internal structure to attain the above aims.

### Introduction

Basically, Frangipani consists of two layers, one is *Petal*, a distributed storage service which provides incrementally scalable, highly available, automatically managed virtual disks mechanism, and the other one is *Frangipani* file system with distributed lock.

Frangipani offers a set of features:

1. a consistent view of files for users.
2. More servers can easily be added to an existing Frangipani clusters.
3. Low human concern is needed in adding servers.
4. A full and consistent backup file system.
5. Tolerate and recover without halting normal operation.

BTW, Frangipani is based on a single administrative domain with trusted network, but it can also be exported to a normal clusters.

### Structure

#### Components

Basically, *Frangipani* uses a two-layer structure.

The lower layer is *Petal*, providing a highly available and scalable system, but petal is just a disk-like system, having no provision for coordination or sharing the storage among multiple clients. Together with Petal, the lower layer includes also a lock service to hold consistency of whole system.

*Frangipani* is the higher tier, providing a file system layer that makes Petal useful to applications while retaining and extending the origin metrics.

Frangipani will be used as standard operating system call interface, and be registered in the system as File system. So, the system can use it just like a normal file system. When writing disks, the content will temporarily set into kernel buffer and wait a *sync-like* operations. Each Frangipani server do not need to coordinate with others and the petal will take the responsibility.

#### Security

The Frangipani servers locate in each machine are beneficial to workload balance, but will bring problems in *security*, since the server is self-authenticated.

To attain security, the servers would better to running in a cluster with private network and trusted operating systems.

However, we can still access Frangipani with untrusted through remote access, since the Frangipani is just a filesystem from the remote view.

### Disk Layout

Basically, the disk layout is only relevant to *petal*.

For *Petal*, it uses a $2^{64}$ memory space as virtual space. And the disk layout is typically split into six parts.

![image-20220419095059932](imgs\Petal Disk Layout.png)

1. Configuration parameters and Housekeeping information

   Although there are 1TB space to keep the information, in fact, only a few kilobytes of it are uesd.

2. Los

   Each Frangipani server use *WAL* mechanism to write metadata. In the figure, the total 1TB space is split into 256 blocks, which shows only 256 users are supported. But this can be easily modified.

3. Bitmaps

   The section is used to describe which blocks in the remaining regions are free. Each Frangipani server locks a portion of the bitmap space for its exclusive use.

4. Inodes

   Each file needs an inode to hold its metadata. The block size of each inode is set to 512 Byte, which is exactly the size of a physical disk block for avoiding to unnecessary contention between servers that would occur if two servers needed to access different inodes in the same block.

5. Small blocks

   The small data block size is 4 KB for each. For any file, the first 64KB will be stored into 16 small blocks, and if there are still data left, other large blocks will be allocated to this file.

6. Large blocks

   The large data block size is 1TB.

To sum up, this kind of design do suffer from fragmentation problems. But the paper believes that, contrasted with the simplicity this design brings, it is still a worthy deal.

And this disk layout also limits the size and number of files, but this limitation can be easily modified by changing block size.

### Logging and Recovery

Frangipani do not use any measures to keep consistency or fault-tolerance for keeping users' files, which is unreasonable at first glance but identical to UNIX file system exactly.

Frangipani provides recover service for metadata through *WAL*, which also improves the performance. When changing file, the metadata change will first write into server in-memory WAL, and wait to be requested by Petal periodically. Write can also be explicitly written synchronously with *sync* command, but latency will increase.

For each server, Log space is limited in size, so a circular queue-like system is applied in logs. When the log space is full, the already committed(i.e. update in logs have been implemented already) logs will be deleted and release space for new logs. 

If any server fails, Frangipani only needs to restart a server with metadata and replay uncommitted logs. However, there are still some details:

1. Updates requested to the same data by different servers are serialized. 

   This can be implemented with write-read-lock

2. Recovery applies only updates that were logged since the server acquired the locks that cover them, and for which it still holds the locks.

   (Like exactly-once guarantee ?) This can be implemented through version number in logs to recored metadata.

3. Frangipani avoids this problem by reusing freed metadata blocks only to hold new metadata.  

4. Finally, Frangipani ensures that at any time only one recovery demon is trying to replay the log region of a specific server.   

### Sync, Cache Coherence and Lock service

Like all distributed systems, Frangipani also needs to attain a trade-off between coherence and concurrency. In frangipani, *a Read/Write Lock* is implemented to this usage. At the meantime, Frangipani will cache data for better performance, so the cooperation between Lock and cache should be emphasized.

A server holding a read lock can read and cache data from disk. But when the server is asked to release its read lock, it must first invalidate the cache entry. And for a server holding a write lock, it must write the dirty data to the disk first before release the write lock.

There may some issues in the details:

1. Is it possible to bypass the disk writing and communicate with other Frangipani server directly?

   It is a good idea to bypass the disk write, and allow other servers directly connect the server holding dirty data to get the newest data.
   But new issues related to fault-tolerance have to be considered. For example, if a server with dirty data fails, the dirty data may exist in several servers.
   For brevity, in Frangipani, connecting directly with other servers are not allowed.  

2. How to prevent the lock service being the bottleneck of whole system?

   The on-disk structures are divided into logical segments with locks, and a single disk sector only can hold one data structure that could be shared. This mechanism keeps the number of locks as few as possible to avoid lock contention.

   - ==why== the small number of locks are beneficial to avoiding lock contention ?

     As far as I am concerned, the small number is not the reason, but also a result. It is still the split mechanism that leads to small number of locks and few lock contention.

     In frangipani, typically, each log is a single lockable segment, and each file, directory, or symbolic link is a dependent segment.

3. How to avoid deadlock?

   Where there is a lock mechanism, there is possibilities of deadlock.

   Frangipani avoids deadlock by globally ordering these locks and acquiring them in two phases.

4. More concretely, How to implement the lock service and deal with lock service failure?

   Basically, the lock service uses a lazy method, which means, a server will hold the locks until others explicitly request the lock.

    Like most systems, Frangipani uses *lease* to detect client failure. Each lease has an expiration time, currently set to 30 seconds after the creation or renewal. Like all distributed, Frangipani also can not figure out the difference between network failure and sever failure.

   If the failed server hold some dirty data, the Frangipani turns on an internal flag that causes all subsequent requests from user programs to return an error.

   In the paper, the authors consider three types of lock service implementations:

   1. A single, centralized server

      A single and centralized server has low latency but suffers from failure, although lock server crash has little effect on recover, but all system may experience a performance glitch.

   2. A pental virtual disk

      A pental virtual disk can survive failure but has a poor performance in normal execution.

   3. Fully distributed for fault tolerance and scalable performance.

      The final way consists of a set of mutually cooperating lock servers, and a clerk module linked into each Frangipani server. The lock service organizes locks into *tables* named by ACSII strings.

      Each file system has a table associated with it. When a Frangipani file system is mounted, the Frangipani server calls into the clerk to open the corresponding lock table. And the lock server gives the clerk *a lease identifier* on a successful open to identify all sequent communication between them.

      The basic message types that operate on licks are *request*, *grant*, *revoke*, and *release*. The clerks and lock servers communicate via *asynchronous messages* rather than RPC to minimize the amount of memory used and to achieve good flexibility and performance. *request* and *release* message are sent from clerk to the lock server, and the *grant* and *revoke* message types are sent from the lock server to the clerk.  

      In this case, the system needs additional measure to handle lock server failures. Some metadata, including a list of lock servers, a list of locks that each is responsible for serving, and a list of locks that have opened but not yet closed each lock table, are kept through Paxos algorithms to reach consensus. Furthermore, if some lock servers fail or new servers are added into the clusters, a two-phase reassignment is implemented:

      1. lock servers that lose locks discard them from their internal state.
      2. lock servers that gain locks contact the clerks that have the relevant lock tables open.

   4. How Frangipani deal with *network partition*?

      In general, the Frangipani system tolerates network partitions, continuing to operate when possible and otherwise shutting down cleanly. As long as a majority of the Petal servers remain up and in communication.

      If a Frangipani server is partitioned away from the lock service, it will be unable to renew its lease.

      If a Frangipani server is partitioned away from Petal, it will be unable to read or write the virtual disk.

      However, there is a hazard here. Supposed a frangipani server send a write and the message delay since network issues, when the message arrive at the Petal, the lease may expire and a new lock is held by another Frangipani. However, since the Petal do not check the lease validation, the stale write are receipt.

      Currently, a 15 seconds error margin are set to try the best to avoiding these case, but still can not ban theoretically.

      An expiration timestamp on each write request to Petal or cooperating between lock server with Petal can be considered.  

### Change cluster members

Since the whole system can be decoupled into two tiers and each Frangipani server do not communicate with each other, adding or removing servers in Frangipani clusters can be more than easier.

A new server can be added by just opening and instructing the Petal disks.

An existed server can be removed by just shutting down, and if possible, flush dirty data are encouraged to do.

### Backup

With *Petal*, Frangipani can use a read-only snapshot for storing the global state. The implementation uses
copy-on-write techniques for efficiency.  

However, when Petal servers fail and want to recover from *snapshot*. They have to retrieve the snapshot which is actually not in the newest state. A Frangipani server need to attain the snapshot and replay logs or write dirty data.

To avoid the above questions, we can set a barrier, which can block Frangipani servers and flush all dirty data, in that case, the snapshot can be newest and need to do no replay. 



## Highlight

The most interesting idea in Frangipani is **Cache** and **Lock**.

The usage of lock server can provide basis of *failure recovery*, *Cache coherence*, and *distributed transactions.* Also, the cache mechanism with lazy write in both data and metadata(WAL log) can hugely reduce the network IO.

However, in current big data system, the lock and cache are not the main focus, that's why Frangipani is considered as interesting instead of useful. 



