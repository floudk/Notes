### Motivation

Traditional batch processing always decouples the state from computation, making users take responsibility of managing state.
The defects in the following mechanism including:

1. Offload the data consistency to application logic, and users need connect to DB to get and manage states.
2. Since the frequent connection between computation and DB system, it's likely that the network could be the bottleneck. 
3. When dealing with partial failures or change UDF, the DB state management mechanism could be troublesome.

Moreover, the DB system forces different teams to use the same database, not decoupling the different jobs, what's more, currently event-driven systems are increasing, the batch system are not suitable any more when confronting with new applications.

That' why we need *Streaming process*.

### Introduction

At the beginning of streaming process, there are two main ideas. 
First, since batch system do not offer real-time process, we can sacrifice accuracy to get real-time results, and later use the batch-processing to update the approximate results.
Another kind of methods utilize batch-processing to imitate streaming process, such as Micro batch processing in Spark.
However, in the above methods, the state management still use the traditional mechanism.

Flink, as the third generation stream processing system, closely integrates state management with computation. *The state is the crown of Flink, and the snapshot is the brightest jewel in the crown.* With special snapshot mechanism, Flink can use the state to recover and reconfigure.

Based on *Chandy-Lamport* mechanism, Flink's snapshot mechanism can deal weak DAG and keep theoretical minimal state. The snapshot mechanism should offer a consistent view to computation and do not halt the normal execution of tasks.

### Core Concepts and Mechanisms

Based on the Dataflow model, the system can be defined as a DAG $G=(\tau,\epsilon)$. However, there is still some difference among DAGs in different phases, such as: *User defined DAG*, *system optimized DAG*, and *physical deployment DAG*.

#### Managed state

State is a main building block of a pipeline as it encapsulates, at any time, the full status of the computation. 

Basically, the state can be scoped into tow types: **Keyed-state** and **Operator-state**.

The state are managed by runtime of the system. With state, the runtime need guarantee exactly-once, and offer a consistent view for the system.

1. **keyed-state**:

   In the most general case, Flink allows for a user to map any record from schema domain S to a given key space L via function *keyby*.

   Flink exposes some API to let user-defined functions collect state.

   - *ListState*: can be used for append state.
   - *ValueState*: can be used to update state.
   - *ReduceState*: can be used to do one or two-step distributive function aggregations.
   - *MapState*: can support put and get k-v operation

2. **Operate-state**:

   Operator state is used when part of a computation is only relevant to each physical stream partition, or simply when the state can not be scoped by a key.

#### State Partitioning and Allocation

1. Physical Representation:

   For a logical graph G, when we want to practically deploy the logical graph, we need to map the logical task to practical containers up to the parallelism.

2. Key-Groups

   For tasks that have declared managed keyed state, it's important to consistently allocate data stream partitions or reallocate in the case of reconfiguration.

   Flink use a hash function and a maximum parallelism Π-max to map user-defined space K into *key-groups*: $K*=h(k) mod \ \ \ (pi-max)$
   Mind that in this method, there could be a trade-off about whether to use index in checkpoint. If we do use index in checkpoint, that will increase the overflow of collecting and network IO. 

   However, if we do not use the index, when we want to get state, we may need to read all states to check.

   For Flink, it use a substantial compromise. Flink use a large enough for coarse grained sequential reading as index and only read if need.

3. State Re-Allocation

   When re-assigning states, the core costs may occur in seek. So we want keep the states as sequential as possible.

   For keyed-state, each instance receives a range of key-groups from $\lceil i·\frac{\pi-max}{\pi} \rceil$ to $\lfloor (i+1) \times \frac{\pi-max}{\pi} \rfloor$.

   For operator-state, each entry is persisted sequentially.

#### Pipelined Consistent Snapshots  

Flink uses **marker** to split the stream data into epochs. A snapshot of the computation at epoch n should includes all handle processes from beginning up to epoch n.

Flink uses ABS snapshot to keep a complete state without halting the normal execution. During this period, the operator needs to do alignment, which is actually local "halt" instead of effecting the whole system.

The main assumptions:

1. Source is replayable.
2. Directional data channels between tasks are reliable.
3. Tasks can trigger a block or unblock operation on their input data channels and a send operation.

For a directed Acyclic graphs, the marker can easily deal with the snapshot:

1. Receive the marker of any input channel and block it until all input channels makers come.
2. Broadcast markers
3. do snapshot
4. return normal

Some operations can be executed concurrently, so it can reduce latency.

However, for dataflows with cycles, the above operations are not satisfied again, since the cycle will cause unlimited running.

Here in dataflow with cycle, we introduce two concepts: *IterationHead* and *IterationTail*:

1. IterationHead receive a maker as source, and begin to block and buffer the cycle edge.
2. Wait all makers comes from inputs, the maker do like other makers and broadcast to the down-stream operators and Iteration Tail
3. When Iteration Tail receive the maker, the buffered data can be considered as kind of state.

Essentially speaking, for dataflow with cycles, we need not only keep the operator states, but also keep the data in the self-looping circle as state, so that the a complete state can be accessed.

#### Usages and Consistent Rollback

Snapshots are not only used for recover from failure, but also can be used to reconfigure, including application logic updates and application version updating.

Also, a replayable source is needed for retrieving the exact state of whole system

### Implementation and Usage

Flink maintains a rich ecosystem of backends, connectors and other services to build a relatively complete Flink ecosystem.

#### State Backend Support

Flink *snapshot* algorithm is used to manage state consistency, but it is also necessary to handle state access and snapshotting for respective state partitions, so we need *state backend*.

Basically, the state backend can be split into 2 types:

1. **Local State**:

   Local State backend can be in-heap or in embedded database system (RocksDB). Both of them can support asynchronous and incremental snapshotting, and since all states are in local, so there is no need for distributed or transactional coordination, and support high-speed(i.e., in local) asynchronous and incremental snapshotting.

   But the memory size or storage size may become the bottleneck of local state.

2. **External State**:

   State can also be accessed with external DBMS, in *Non-MVCC* or *MVCC*.

   > MVCC: *Multiversion concurrency control* is a method to implement concurrency control.
   > The basic idea to implement concurrency control is use a lock to control read and write, but it is not enough to satisfy high concurrency visit. So the concept of *R-W lock* comes in, and this avoid lock among read operations. 
   > Furthermore, Is there a method to let write be concurrent? *MVCC* uses a multiversion method to use temporal replicas to enable concurrent write operations.

   - **Non-MVCC**: For non-MVCC database, it can use a 2-phase commit process to store a snapshot. With WAL, changes are put into the transaction log(WAL)  and pre-committed when *triggerSnapshot()* is invoked. Once the global snapshot is complete, pre-committed states are fully-committed by the JobManager in one atomic transaction.

   - **MVCC-enabled**: MVCC-enabled databases allow for committing state across multiple database versions.

     The main benefits of external backends is that:

     1. Rollback recovery does not need any I/O to retrieve and load state from snapshots.

        > Why does not rollback recovery need any I/O?
        >
        > It seems any existed Flink Docs do not mention this *external state*, and it is said to be not materialized to source projects.
        >
        > However, as far as I am concerned, the meaning of not need any I/O to retrieve from snapshot is that when using a local state backends, we still need to make states persistent, so it is necessary to use a DFS system to do snapshot, and retrieve the state from snapshot when needed. But with a external state management, the snapshot is exactly the state itself and do not need another snapshot anymore.

     2. All external backends is support for incremental snapshotting.

#### Asynchronous and Incremental Snapshots

> *Copy-On-Write* (COW): Modifications must still create a copy, hence the technique: the copy operation is deferred until the first write. 
>
> *LSM tree*: a data structure with performance characteristics that make it attractive for providing indexed access to files with high insert volume, such as transactional log data. The basic idea of LSM-tree is sequential writing in disk can be much better than random writing.

The snapshot is actually a copy of current system. But actually, the copy is not required to be physical, it can be just a logical snapshot that is *lazily* materialized by a concurrent thread, like a *copy-on-write* data structures.

More concretely, Flink's local backends allow for asynchronous snapshots are as follows:

1. *Out-Of-Core State Backend*: Updates are not made in-peace(meaning that the new snapshot replaces the older one), but are appended and compacted asynchronously.

   With version number, the operator can then continue processing and make modifications to the state. An asynchronous thread iterates over the marked version, materializes it to the snap shot store, and finally releases the snapshot so that future overwritten can be executed.

2. *In-Memory Local State Backend*:  The in-memory state backend is based on hash tables.

   During a snapshot, it copies the current table array synchronously and then starts the external materialization of the snapshot, in a background thread.

   The main thread of stream process does lazily copy and will overflow chains upon modification if the background thread is still running.
   **Incremental snapshots** are possible and conceptually feasible, but still not implemented at this point.

There is another feature which can trigger notifications about completed snapshots back at the tasks that request them. And this is especially useful for garbage collection, discarding WAL and so on.

#### Queryable State

Flink decides to support external Queryable state based on the following observations:

1. Some clients need ad-hoc access to the application state for faster insights.
2. Flink output to external systems may frequently become a bottleneck to the application since the low read-write rate to external systems.

For a client owning the right to do read, explicitly deployment Flink do answer that query asynchronously 

#### Exactly-Once Delivery Sinks

Excepts consistency, the side effect of system also need to be noticed, especially *exactly-once guarantees*. 
For Flink, the internal state can guarantee exactly-once with snapshot mechanism and replayable source, but the sink need extra measures to keep the exactly-once.

1.  **Idempotent Sinks**: An idempotent operation means that executing it once has the same result as executing it multiple times.

2. **Transactional Sinks**: But idempotent is not always satisfied, when idempotency can not be guaranteed, committing output to external systems has to be made in *transactional way*, just like database systems.

   Typically, the transactional way could be:

   - **WAL Method**: Based on Write-ahead-Log, the sink operator will keep the data as state. Only by receiving checkpoint, the in-memory state could be "flushed" into external components.

     This method is easy to implement and has roughly no limitation to external components, but the in-memory states are not always reliable.

   - **2PC Method**: The sink will write the data as *pre-committed* to the external system, and only when receiving the checkpoint, sink will formally commit the data to the external.

     This method can guarantee exactly-once in all cases, but needs external system support the 2-PC phases.

#### High Availability & Reconfiguration

Since all metadata is in *JobManager* node, the node are facing a single-point-of-failure situation. 

Flink use a *passive-standby high availability scheme* to undertake the coordination role of the failed master node via distributed leader election with Zookeeper.

This of course introduces some additional latency for critical operations, however, for non-critical operations, the commit can be asynchronous.

### LARGE-SCALE DEPLOYMENTS  

Based on large-scale deployments, it is interesting to discusses the following questions:

1. What affects snapshotting latency?  

   Since the snapshot is asynchronous, so the latency does not effect the system. And the fundamental latency comes from *alignment* time.

   But the alignment time is not effected by global state size, so the latency roughly has no effect by how much state is snapshotted.

2. How and when is normal execution impacted?  

   Still, the alignment will use block on input channels, so more connections can introduce higher runtime latency overhead.

   More concretely, the overall alignment time is proportional to two factors:

   1. Total alignment times: shuffles number or total operators number
   2. Time of one alignment: the parallelism of the tasks.

   However, in large-scale system, this kind of latency has little effect on the system.

### Related Work

#### Reliable Dataflow Processing

1. IBM Streams employs a pipelined checkpoint mechanism.

   However the *snapshotting protocol* is different from Flink approach

   1. all records in transit are consumed in order to make sure that they are reflected in the global state while blocking all outputs.  
   2. All operators trigger their snapshot in topological order, using markers as in our technique and resume normal operation.  

2. Naiad’s proposed three phase commit disrupts the overall execution for the purpose of snapshotting.  

#### Microbatching

1. Stream
2. Trident  

