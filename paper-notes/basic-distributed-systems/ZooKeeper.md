## ZooKeeper: wait-free coordination for Internet-scale systems

ZooKeeper is kind of framework to simplify distributed system development, including leader election, metadata management and group membership management, and ZooKeeper is just the fragment to do this jobs.
Basically, ZooKeeper offers some primitives (API) for clients, and clients can use this API to do much more complicated work, including configuration management, distributed lock, namespace management and cluster management.

In a sense, it can be considered that **ZooKeeper = Unix filesystem + Watch mechanism **

### Service

> ZooKeeper provides to its clients the abstraction of a set of data nodes (**znodes**),  organized according to a hierarchical namespace.

Clients can use supplied API to operate these znodes to implement their own distributed services.

#### znodes

znodes can be split into two kinds:

1. *regular* znodes: nodes created or deleted by clients explicitly.(node is persistent after a session )
2. *ephemeral* znodes: nodes can be deleted or created by both clients explicitly and system automatically.

When creating a new znode, a *sequential number* can be set, which is increasing according to the order of creation.

#### data model

znodes are *not designed for general data storage*, but can be used to store some info about metadata, configuration, and version information in a distributed computation.

#### watch trigger

A watch trigger can be used to do :

1. Monitor data changes in znode.
2. Monitor znodes creating or deleting.

#### Client API

1. *create(path,data,flags)*: create one znodes storing data[] in path, and flag is used to denote the node type (regular or ephemeral)

   return the name of the new znode.

2. *delete(path, version)*: Delete the znode in path iff it is in the given version exactly.

3. *exists(path watch)*: Return true if the znode exists in the path, and watch flag enables the client to set a watch on the znode.

4. getData(path,watch): Returns the data and meta-data. watch flag is as above

5. setData(path,data,version) Write data[] in the znode path iff the version is the current version of the znode

6. getChildren(path,watch): Returns the set of names of the children of a znode.

7. sync(path): Waits for all updates pending as the start of the operation to propagate to the server.

#### ZK guarantees

ZK offers two basic guarantees:

1. Linearizable writes: all write operations are serializable. Notice that, the Linearizable is actually *A-linearizable*, which means a clients can have multiple outstanding operations.
2. FIFO client order: all operations from a given client are executed as the FIFO order.

#### Higher-level services

1. **configuration management**

   The system can store the configuration in a znode(called $z_c$), and set watch flag in the node.
   So every server can use the local configuration cache to run, and if the configuration is updated they all can receive triggers.

2. **Rendezvous** (metadata management)

   Some meta-data can be stored in a given node,especially the information temporarily specified by the system

3. **Group membership management**

   Since the ephemeral is created and deleted automatically, so it also can be used to detect the state of a session.

   So, we can just let each member create a node in a given path, and when it fails or quits the node is just deleted, and if we want all members in a given group, we can just list the children.

4. Lock

   1. simple lock

      Just use a znode in a given path. A client will try to create a ephemeral znode to obtain the lock, if succeeds, the lock is obtained, or a watch will be set to inform the deleting of the znodes so it can try it again.

      However, this suffers as least two problems: herd effect and only mutual lock. Herd effect means even a znode can be owned by one server, every time a process give up the znode, it will informs all servers setting watch flag in the znode; mutual lock is kind of inefficient for read operation do not need to be mutual. 

   2. simple lock without herd effect.

      We can do a little modification to the above process.
      All clients try to obtain the lock will set a child in the Lock znode sequentially, and only the earliest (at lowest index ) can get the lock, or it will set a watch in the znode just proceeding the current.

      So there is no herd effect.

   3. R/W lock

      Since read operation is not necessary to be mutual, when setting the znodes in client's requesting, we do different operations in read and write.

      For write operations, it must be mutual, so it can be identical as the above operations

      For read operations, if no lower write znodes, it can get the lock, or set watch in the write znode ordered just before n.

5. double barrier

   Set two sync point, so clients can synchronize the beginning and end of some computation.

   When a process are ready to enter a barrier, it set a child in a given znode, and all related znodes are not allowed to continue unless the children number exceeds the predefined thresholds.

### Implementation

ZooKeeper is a distributed system itself, so it is necessary to provide high availability, including fault-tolerance and or so.

For better real-time and simplification, ZK maintains a cache in all servers, and all read operations are handled with local state, and only write operations need coordination among clusters. This local read may cause stale return, but not all applications are strict for this, but if needed, use sync to guarantee the newest value.

The ZK will maintain a replicated database in memory containing the entire data tree. Each znode in the tree can store maximum 1MB data(can be changed in configuration). When writing the data to in-memory database, we first update to disk, and keep a replay log and snapshot in disk to recover. ZK cluster also consists of leader and followers, especially for write operations.

Basically, for an op need coordination, a request will experience *request processor*, *Atomic Broadcast*, *Replicated database* three phases.

1. Request Processor

   The processor will transfer the request from clients to be transactions, which is idempotent. Hence, even for servers apply more transactions than others, the local state never diverge.

2. Atomic Broadcast

   All requests that update ZooKeeper state are forwarded to the leader, and the leader executes the request and broadcasts the change to the cluster through Zab, an atomic broadcast protocol.

   Zab by default uses a simple majority quorums to decide on a proposal, containing **election, discovery, sync and broadcast** four phases. By default, the Zab uses *fast Paxos* algorithms. And Zab further guarantees the change broadcasts are delivered in the order they were sent and all changes from previous leaders are delivered to an established leader before it broadcasts its own changes.

3. Replicated database

   Each replica has a copy in memory of the ZooKeeper state, so after recover from failure, the server need to rebuild the state. With snapshot, the server can only replay few logs to rebuild the database.

   The snapshot is kind of **fuzzy snapshot**, which means we do not set a lock when snapshotting. That's to say, we do a depth first scan of the tree to record data and meta-data and write to the disk, however, the state may not be identical to any state of ZK. But since the state change is idempotent, multi-apply is safe if they are applied in order.











### Evaluation

the system are evaluated through throughput, latency and barriers performance.

Particularly, the throughput are evaluated in read/write and some failures cases.



