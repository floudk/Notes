### Motivation

Currently, a lot of commercial platforms scarify strong consistency to guarantee high availability and high performance. But this trade-off is so inflexible.
And Chain Replication are proposed to offer  both high-performance and availability and strong consistency. However, CR still suffers from some problems.

1. All read operations come into tail node, causing tail node be a hotpot.
2. In multiple CR, It's hard to keep workload balanced with hash algorithm, especially in practice.
3. When implemented among datacenters, the tail node may be in far-extent 

To overcome the above problems, this paper proposed **CRAQ**, i.e., Chain replication with Apportioned Queries.

### Introduction

CRAQ is kind of object-based storage, unlike the filesystem storage. Object-based storage owns flat namespace, and do not need to take complicated hierarchical operations.

Based on Chain Replication, CRAQ mainly allow read operation come not only for the tail, but also for all other node, which can increase system read throughput and latency in high degree. At the meantime, CRAQ also take implementation into consideration. While read a object, the system will select the local node to respond.

### System Model

Basically, CRAQ is inspired from [CR](.\Chain Replication for Supporting High Throughput and Availability.md), so the basic system components are almost identical to CR.

And the main optimization is that in CRAQ, every node, instead of only tail in CR, can respond to a read operation, which theoretically can improve read throughput to (length-1) times and also can improve write-heavy operations.

To make every server available to read operation and do not scarify the consistency, CRAQ introduce the version concept. Each server in the chain should store multiple versions of a object and a state indicating clean or dirty of the object.

A write operation will pass along the chain, and for a server in the chain receiving the write operation, it will store and mark it as *dirty*, which means uncommitted. When the tail receive the write, the tail will store it and mark it as clean, after that, the tail will return an ack to other servers backwards the chain.
After a server receiving an ack, it can mark corresponding version to be clean and delete all objects whose version number is earlier.

If a server receive a read request, it will do the following behavior:

1. if the newest object is clean, it just respond.
2. if the newest object is dirty, it should call tail to query the newest committed version, and decide which version to respond.
   Notice that, the version query is much cheaper than complete response, so it is also beneficial for write-heavy operations.

Besides that, the CRAQ also supports weaker consistency guarantees.

1. Eventual Consistency   

   That's to say, all server respond read only according to local state, i.e., server will return the newest clean state. Here, eventual means the consistency will be guaranteed only after being committed.

2. Eventual Consistency with Maximum-Bounded Inconsistency  

   In this case, server also do not need any coordination for respond any read operations. However, we can set a bound to let the server use dirty state to respond, such as time limitation or version limitation. 

### Scaling CRAQ

In this section, we main focus on the question that how to place CRAQ within one or multiple datacenters, and how to manage the meta-data of CRAQ ?
To understand why these question matter, it is necessary to list some special cases:

1. Almost all write operations gather in one datacenter.
2. Almost all operations in a datacenter only have relationships with part of the datacenter.
3. Objects needs different replicas according to popularity.

#### Placement strategies

When Placing chains across multiple datacenters, the strategies can be flexible in CRAQ. It offers multiple methods to support different deployment mechanisms, such as:

-  Global chain size & Implicit datacenters

  `{num_datacenters,chain_size}`

  in this design, the total number of datacenters are determined but which to choose is not specified. Consistent hashing are used to determine which datacenters store the chain. And all datacenters should use the global chain_size as configuration.

- Global chain size & Explicit datacenters

  `{chain_size,dc1,dc2,...,dcn}`

  In this case, the datacenters used to store is specified. Further, to decide which nodes in a datacenter to store the chain, still, the consistent hashing method makes difference.

  Also, chain_size are used in all datacenters as configuration.

- Explicit chain size & Explicit datacenters.

  `{dc1, chain_size1, dc2,chain_size2,....dcn,chain_sizen}`

Besides these methods, more complicated methods are also supported.

#### CRAQ within a datacenter

CRAQ can use the same method as CR with more meta-data are needed to store.

#### CRAQ Across multiple datacenters

When considering multiple datacenters to deploy CRAQ, the network topology should be taken into consideration to take advantages of locality.

Simply speaking, CRAQ could use topology information to make read and write operations all achieve the smallest possible latency.

#### ZooKeeper coordination service

ZK can be used to manage the coordination service, including metadata management and or so. However ZK is not able to support above operations based on topology.
To make ZK support it, we can use a two-level hierarchical ZK. the top level is a global ZK to do coordination and the lower ZK to do local management. Or we can improve ZK, adding topology information to ZK so it can support this.

### Extensions

In this section, two aspects of extensions are proposed to enhance the performance of CRAQ.

#### Mini-transactions 

The mini-transactions aims to expand the native CRAQ capabilities in a lightweight way, so that CRAQ can support more diverse operations, including permissions and directory changes, etc.

Mini-transactions can be roughly divided into three types:

1. Single-Key operations

   - Prepend/Append
   - Increment/Decrement

   - Test-and-Set

   Native CRAO is roughly enough to support single-key operations.

   The former two operations are simple in CRAQ, since only head is involved in. CRAQ can change the newest version in head (no matter whether it is clean) and consider it as a new write operation to transfer. Notice that the head can buffer these operations to reduce the coordination and improve throughput. 

   For test-and-set, CRAQ can receive a version number(also the head), and check whether the newest object version are identical.
   If the newest version is not clean, CRAQ will either return error to client (i.e., the client are responsible for the error) or just block and wait until it is clean. However the latter method will effect the performance in a high degree.

   Other method are also supported by CRAQ, such as value check or so, however, the consistency guarantee are always needed in all cases.

2. Single-Chain operations

   A minitransaction is defined by a *compare*, *read*, and *write* set; Mini-transactions use a two-phase method to do that. Firstly, it tries to lock a set of addresses, and if success, it commits the command, Otherwise, it will releases all locks and retries later.

   However, when doing that in CRAQ, two-phase is not necessary, since only one server- head is involved in, which can improve the performance of mini-transactions in CRAQ.

3. Multi-Chain operations

   Like single-chain, also only heads of multiple chains are involved in, which reduces the complication of transactions. Notice that the block operation will always reduce CRAQ performance.

#### Multicast

Multicast can be used in both head's write and tail's ack. Through this way, only little meta-data are needed to transfer across to the chain (in ack process, even no meta-data is needed).

If some interval server do not receive the info, it's ok to query the processor(in write) or tail directly (in ack).

Notice that there is no ordering or reliability guarantees in both process. 

### Management and Implementation

This paper mainly describe three points:

1. CRAQ need ZK to maintain membership info.
2. The connection between each server is implemented with TCP and DHT.
3. To enable add server in any position of a chain, CRAQ also support back-propagation to transfer state.

### Evaluation

Based on the purpose of CRAQ's design. The evaluation roughly focus on three topics:

1. The performance of CRAQ in Read-only systems.
2. The performance of CRAQ in Write-heavy systems.
3. The wide deployment of CRAQ and the performance effect.
