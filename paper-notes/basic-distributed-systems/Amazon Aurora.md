# Amazon Aurora: Design Considerations for High Throughput Cloud-Native Relational Databases  

## Motivation

Traditional databases confront with three main issues: *availability, fault-tolerance, and schema modification*. To solve the questions mentioned above, with the advent of developing cloud infrastructure, a lot of databases based on cloud are currently *decoupling compute from storage* and *replicating storage across multiple nodes*.

However, new issues are coming instead. The above measures greatly reduce IO pressure in disk, but make the *Network* be the new bottleneck, since all operation should be transferred through network, and the distributed AZs and parallel communication deepen this trend. 

Furthermore, traditional operation in database also introduce stall and context switch, causing high latency. For read operation, if there is a cache miss, the read op should wait until disk IO finished, which may cause flush when the memory is full. For write op (transactions), traditional MySQL use *2-phase commit* model, which is a great burden on the network and considering the common mistakes in software and hardware in cloud environment, 2-phase commit introduces much more useless IO needs, making costs outstrip benefits.

## Introduction

To address the issues, Amazon proposes **Aurora**, which aggressively **leverages the redo log** across cloud environment.

> **Logs** are an important part of *MySQL* system, including error log, transaction log, and or so. **redo log** and **undo log** are both components of transaction log.
>
> An *undo log* is a collection of undo log records associated with a single read-write transaction. An undo log record contains information about *how to undo the latest change by a transaction* to a clustered index record. If another transaction needs to see the original data as part of a consistent read operation, the unmodified data is retrieved from undo log records.
>
> The *redo log* is a disk-based data structure used during *crash recovery to correct data* written by incomplete transactions. During normal operations, the redo log encodes requests to change table data that result from SQL statements or low-level API calls. Modifications that did not finish updating the data files before an unexpected shutdown are replayed automatically during initialization, and before connections are accepted. For information about the role of the redo log in crash recovery,

Aurora separates some functionality from the kernel and reduce communication need.

Contrasted with traditional approaches, Aurora has three benefits:

1. Database storage, as an independent distributed storage service, can protect the database from performance variance and transient and network or storage failures.
2. Greatly reduce network IO by only writing redo log records to storage.
3. Decouple some expensive functions from kernel to storage, system can asynchronously execute to get a high-performance.

And three main contributions are described in this paper:

1. How to ensure the consistency of the underlying storage based on the **Quorum** model?

   > Quorum model: A *quorum* is the minimum number of votes that a distributed transaction has to obtain in order to be allowed to perform an operation in a distributed system. 

2. How to push down **redo log** related functions to the storage layer?
3. How to eliminate synchronization points, do checkpoints and failure recovery under distributed storage?

> **Transaction**: A way of wrapping multiple operations on may different pieces of data and declare in that that's entire sequence of operations should appear *atomically* to anyone else who is reading or writing the data.

## History

- EC2: EC2 is roughly the origin format of cloud service. Amazon offers a virtual machine with local disk to clients. 

- However the defects is obvious, the server is closely coupled with local data. That's to say, if a server fails, the data is also offline.

- EBS: To solve the above questions, the Amazon offers EBS decoupling the data and servers. A EBS server is an abstraction of disk. For clients, if a DB server fails, they can just start a new DB server connected to EBS server, as a result, the service still works.

  The defects could be the fault-tolerance question. Basically, Amazon offers a chain-replication way to offer reliable service, which is not ok if a whole *AZ* fails.  

- RDS: To solve the above question, the Amazon tries to deploy replicas into different AZs, but it seems the necessary connection among those replicas could be a huge costs, that's why we need Aurora to reduce the network I/O.

## DURABILITY AT SCALE  

### Quorum-based protocol

**Quorum-based voting protocol** is one of the most important protocols in distributed system, used in Paxos, Raft and *Aurora*, which is the focus in this paper.

Typically, a quorum-based protocol with *V* nodes can handle read and write request based on voting mechanism. For write operation, protocol needs servers $V_w > \frac V 2$ to make a consensus, and for read operation, protocol needs servers $V_r > V - V_w$ to guarantee the data consistency.

### Why 3/2/2 is not enough ?

But based on AWS practical experience, the 2/3 quorum *is not enough* in current environment. Currently, almost all cloud service providers split datacenter into **AZs**(available zones), and each AZ is independent to large-scale failure such as flood, earthquake and or so. Servers in the same AZ is correlated to these large-scale failures.

To adapt these kind of large-scale disasters, it is natural to place 3 servers in 2/3 quorum in three AZs respectively. But when an AZ is unavailable, the 3/2/2 will back to 2/2/2. In this case, it's likely that system can not reach a consensus, i.e., the left 2 replica happen to be inconsistent, hence, system is neither readable nor writeable. Furthermore, there is no fault-tolerance in this system, and another failure in any of the left 2 servers will down the system.  

*Aurora* proposes that every AZs need to store 2 replica, namely, $V=6\ \  V_w=4\ \ V_r=3$. The benefits is that:

1. Can be readable even if AZ+1 fail. That's to say, 6/4/3 will back to 3/4/3 when an AZ and a server fail. And the system should repair to 6 as soon as possible.
2. Can be writeable and readable even if an AZ fail.

### Is 6/4/4 enough?

For the above discussion, we can conclude that 3/2/2 is not enough, *but is 6/4/4 enough*?

Whether a mechanism is enough depends on the possibility that a server failure happen during repairing from the minimal full state(In 6/4/4, the minimal full state is 4 nodes). Terminologically, that's the possibility of MTTF(mean time to failure) during MTTR(mean time to repair). And from the AWS practical experience, the possibility is low enough, and the paper believes 6/4/4 is enough.

By the way, when talking about database replica, it's natural to split databases into segments, and a storage volume is a concatenated set of 6-replica(called *PG*: protected group). Hence, we can just monitor and discuss the segment in this paper. Especially, the segment is 10GB in size in AWS aurora.

### Other benefits from high resilience

For a system resilient to long failures, it is also resilient to short failures. And this kind of fault tolerance is not the only benefit.

Furthermore, based on the resilience, we can do operations including *heat management*(hot-reconfiguration),   *server migration* and so on.

## THE LOG IS THE DATABASE 

### Why not traditional MySQL ?

We have mentioned above that currently the main bottleneck of our system is *Network IO*. And Based on the 6/4/4 quorum protocol, here we try to discussed why traditional SQL is not enough.

For traditional SQL, the typical write operation will invoke update of *redo log, binlog, modified data page, double write content,and metadata*, and the replication of quorum and the storage itself amplified these network pressure.
Furthermore, some of these replication process are sequential, making the latency much longer. 


### Offloading Redo processing

It's easy to find that a lion's share of data are not actually serve for data but for recovery. If we only keep the redo log(of course with metadata) to be transmit through different instances, the network pressure can be better off.

Keeping only redo log, we can say that **the log is the database**. The background storage tier can maintain lot versions of checkpoint and replay the redo log asynchronously.

With this mechanism, we can decrease the network IO and handle much more transactions compared with traditional SQL.

### Asynchrony in front-ground and back-ground

The benefits are not only reducing Network IO, but also decouple the front-ground from complicated sequential work, minimizing the latency of the foreground write reuest.

In traditional SQL, the background writes of pages and checkpointing process will affect the frontground process, but in model posted in this paper, the background writes of page and checkpointing are decoupled from frontground, even the background process can be beneficial to frontground since the read operation can be responded much quickly.

And almost all steps can be executed asynchronously, so for foreground, the latency impact can be minimized.

## The Log Marches Forward 

In this section, we focus on *how the log is generated from the database engine so that the durable state, the runtime state, and the replica state are always consistent.*

### How to keep consistency without intolerant 2PC ?

To begin with, it is worth mentioning the WAL(*write-ahead-log*) mechanism.

> **WAL** is kind of mechanism acting as an interval tier to guarantee *Atomicity* and *Durability* in db systems. For MySQL, the main components could be **redo log** and **undo log**. All write requests will first enter redo log, and then write to database. During the writing process, if something wrong, db system can use undo log to rollback._
>
> *Checkpoint* mechanism here is used to synchronous WAL and database systems and also to reduce memory occupied by WAL.

In aurora, each log record has a monotonically increasing value called **LSN**(Log Sequential Number), and if servers lose some LSNs, they can just use a so-called *gossip protocol* to connect others and fill in the holes. Basically, we use these LSN to mark systems points of consistency and durability, and just continually advance these points as new requests coming.

Under the redo mechanism with *LSN*, system can handle a read request from servers own complete redo logs rather than visiting storage tier with quorum read, which is only needed in repairing process.

During the recovery of aurora, the DBMS could have many independent transactions that are not complete(i.e. finished and durable). The database could use origin(MySQL) logic to decide whether a transaction need to rollback, but this must based on the fact that the storage tier should firstly arrive a consistent state or uniform view. In order to make it, the paper proposes some terms based on *LSN*:

- **VCL**: Volume Complete LSN. For all log records with LSN lower than VCL, the storage tie guarantee that they are available and works. That's to say, when recovering from failure, all logs with LSN bigger than VCL should be truncated.

- **CPL**: Consistency Point LSN. The logs are not independent in aurora and MySQL, the CPL is last log's LSN.  That's to say, we need not only truncate LSN > VCL, but also *truncate LSN > highest available CPL* (called **VDL** volume durable LSN).

  > A **transaction** in MySQL is a sequential group of statements, queries, or operations such as select, insert, update or delete to perform as a one single work unit that can be committed or rolled back. Transaction consists of a group of **mini-transactions**. That's to say, mini-transactions should be considered as the atomic unit in MySQL.

And since VDL is always less than or equal to VCL, so on recovery, the database talks to the storage to get the VDL and then issues commands to truncate all logs above VDL.

### How to do normal operations with only Redo Log ?

#### Write

To advance VDL, the database need to interact with storage service to maintain state and establish quorum. During these processes, there are two points need to be mentioned:

1. **LAL**: LSN Allocation Limit. That's to say, when allocating LSN to a new log, system need guarantee that the gap between newest LSN and VDL should be kept in a certain range, so that, the storage will never be the bottleneck of the whole system.

2. **Backlink**: Since segment mechanism is used in the storage system, every segment only sees a subset of log records. As a result, each log record contains a backlink identifying the previous log record. Backlinks could be sued to track the point of completeness of the log records.

   Based on backlink, we can define **SCL**, Segment Complete LSN, the greatest LSN below which all log records of the PG have been received. The SCL can be used in *gossip* protocol to find and exchange missed logs.

#### Commit

In aurora, the commit is asynchronous. A commit transaction consists of a group of logs with a *commit LSN* (the largest LSN in these logs). The work thread will push it into a persist Queue and handle other events. When system VDL is greater than the commit LSN, system can respond to client. 

In this case, no work will be blocked for committing.

#### Read

In almost all SQL, read will first query cache instead of disk. If cache missing happens, the system will read from disk.

However, in this process, it is likely that the cache happens to be full, and a selected page should be pushed to load a new page. Further, if the selected page is *dirty page*, there must be a sequential operation to flush, which may cause great latency for system.

For aurora, it adds extra constraints so that we do not need to flush anymore. The *constraint* is that, we only throw away page *Page LSN <= VDL*. With this constraint, system guarantees that:

1. All updates in this selected page is already applied in storage.
2. If a cache missing happens, we can use current VDL to get newest page version.

Based on VDL mechanism, normal read operations do not to do quorum read. When a read request coming, the system will generate a read-point based on current VDL. Since the database directly manages all replicas, so it can select a server that owns complete state relative to read-point.

#### Replicate

In Aurora, write replica instances and up to 15 read replica instances share a set of distributed storage services, so adding read replica instances does not consume more disk IO write resources and disk space.

If the corresponding data page is not in the buffer pool when the log is played back, it will be discarded directly. The reason why it can be discarded is that the storage node has all the logs. When the data page needs to be accessed next time, the storage node can construct a specific data page version according to the read-point.

### How to avoid expensive redo processing on crash recovery ?

For most db systems, the failure recovery is based on *checkpoint* mechanism, and is an expensive operation. Since on restart, any given page could contain committed data and uncommitted data, so the system should redo since last checkpoint, causing crash recovery costs are relevant to checkpoint intervals. But *no such tradeoff is required with Aurora*.

Traditional replays in a offline way, but aurora's decoupling storage from database can replay in parallel in the background. So the recovery can be much faster even if in heavy workload. And database need to do a quorum connection to attain a consensus of VDL. As mentioned before, there is a limitation to the gap of allocating LSN and current VDL. That's because after redo replaying, the database also need to undo to rollback the uncommitted logs. And this can be down in parallel too. Database system can inline if the redo logs are replayed.

 ## A bird's view of whole system

Based on what we have discussed above, in this section, we can propose a brief conclusion.

Aurora is based on the community MYSQL/InnoDB branch to develop, so a lot of concepts are almost similar, but still aurora owns some unique optimization to its own usage.

### MySQL

Write operation in InnoDB will write both cache and redo log buffers. When it is ready to be committed, InnoDB will write redo log buffers to disk, and write modified buffer pages with additional double-write operation.

InnoDB also has other subsystems, including the transaction subsystem, a lock manager, a B+- implementation and associated notion of a "mini transaction"(MTR). An MTR is a minimal group of actions which must be executed atomically.

### Aurora

Aurora offloads the disk IO into storage tier. Use the largest LSN of each MTR as consistent point.

The storage instances are relatively independent to database instances. Storage instances tier provide uniform data view for high level database instances.



