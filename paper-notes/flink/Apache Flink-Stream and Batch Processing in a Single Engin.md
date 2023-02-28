### Motivation

Currently( in the time of proposing this paper), most data processing is still batch processing, but the fact is that majority of data originated in a streaming way. Use the batch processing method to process the streaming data is never a sensible method, in which we need store the streaming data to form a batch.

Some methods, such as *Lambda* structure, are proposed to solve this problem, which can do streaming process approximately in real-time and do exact batch process later. However, these methods still suffer *high latency*, *giant  complication* and *arbitrary inaccuracy*.

So in this paper, we present the **Flink** structure, which supports streaming process from the beginning, and also keep the special API for batch process.

### Introduction

Apache is an open-source system for processing streaming and batch data, and can pipeline these processes. There are three awesome points in Flink.

1. A unified architecture of stream and batch data processing.
2. Streaming, batch, iterative and interactive analytics are represented as fault-tolerant streaming dataflows.
3. A flexible windowing mechanism.

### System Architecture

#### Flink Structure model

<img src="imgs\Flink software stack.png" alt="image-20220309134038807" style="zoom:50%;" />

Just like the above figure, Flink consists of basically four components, **Deployment-Core-API-Libraries**. 

- Deployment is the basic tool or environment for running Flink. For different cases, respective tools should be offered to Flink.
- Core is the *distributed dataflow engine*, which executes the dataflow programs. By mentioning *dataflow programs*, it means a DAG of stateful operators connected with data streams.
- API: Like mentioned before, Flink offers two kinds of API, streaming API and batch API. Both can generate programs work well in the core engine.
-  Libraries is task supported tools, consisting of Flink ecosystem.

#### Flink Process Model

<img src="imgs\Flink process model.png" alt="image-20220309134129494" style="zoom:50%;" />

When it comes to processing data with Flink, there are basically three roles: **Client**, **Job manager** and **Task manager**.

- Client

  A client will propose a data processing task, and the job for transforming the process to a DAG is definitely taken by client. Besides, the clients also need to check datatype and do task-specific optimization.

- Job manager
  A Job manager is like a leader, who will not only manager the operators and data, but also need to do checkpoint and handle potential failure of a computation.

- Task manager
  Task manager, like a follower, is the most basic role in a classical Flink process procedure, executing the task and report the status of the task.

### Common Fabric

No matter which kind of API is invoked, all programs will be transformed to **Dataflow graphs**, so it can be executed by the core engine.

#### Dataflow graphs

Basically, a dataflow graph consists of *stateful operators* and *data streams*.

- Stateful operators are parallelized into one or more *subtasks*.
- Data streams are split into multiple partitions to  correspond one-to-one with subtasks.

Between two operators, the data transferring patterns can be various, such as *end2end, broadcast, re-partition, fan-out, and merge*.

#### Intermediate Data exchange

Basically speaking, *the intermediate data exchange process* can be considered  as classical *Producer-Consumer* method.

Generally, Flink exchanges data through pipeline method with buffers for temporary data-peak. However, to reach better system efficiency and other aims, Flink uses additional mechanism.

1. **Data Block**

   Sometimes, it is necessary to break pipeline, including special programs with demands of distinguish data process stage, and batch process(i.e. block a stream to form a batch).

2. **Batch or Timeout Sending**

   Data exchange is implemented with *buffers*. Since we can not send as soon as any commands comes, it causes low throughput to the system. Flink sends a buffer in two cases:

   1. A buffer is full
   2. Or timeout is exceeded.

   In this way, Flink can pursuit a trade-off through latency and throughput.

3. **Control Event**

   The control will be injected into the stream and processed by operators. The operators will do respective behavior when receiving the so called *barriers*. Typically, there are three types of control event

   - checkpoint barriers: divide the stream to do a snapshot.
   - watermarks: offer support to event-time relevant operations.
   - iteration barriers: offer support for iterative algorithms.

   By the way, although control event are in pipeline order, namely, it can be processed in-order, but to avoid low efficiency, the operator will merge all receiving streams. That's to say, Flink never guarantee the ordering guarantees, which should be considered in client.

#### Fault Tolerance

Flink guarantees *exactly-once* execution, and fault tolerance also need to do his job to this aim by *checkpointing* and *partial re-execution*. Here we focus more on checkpointing.

The key of checkpointing is snapshot, an the most challenging point is do snapshot while have fewest effect on normal execution.

Flink use Asynchronous Barrier Snapshotting (**ABS**) to do snapshot. ABS is the implementation of Chandy & Lamport algorithm in Flink. Based on barriers and block mechanism, Flink can do not record any channel state and only care operators state, guaranteeing the storage is kept to the theoretical minimum.

Besides this, ABS also provides other benefits:

1. It guarantees exactly-once state updates without halting the computation.
2. It is completely decoupled from other forms of control messaage.
3. It's completely decoupled from the mechanism used  for reliable storage.

#### Iterative Dataflows

Iterations and incremental processing are necessary to most applications. And Flink offers iterative support by resubmitting a new job or just adding additional node or feedback edge to do the iteration.

### Stream Analytics on top of dataflows

The Flink runtime originally supports streaming relevant operations, so for Stream API, Flink only need to provide another mechanism to implementing a **windowing system** and a **state interface**.

#### Time Notion

Basically, there are two kinds of time concepts in Flink:

1. Event-time : time when an event originates.
2. Process-time: time when processing an event.

There are always some little difference between the two concepts, and Flink use a mechanism called **watermark** to avoid the effect caused by the difference. For a operator, when it receives a watermark t, it means no events earlier than t will arrive from that moment.

For an application using *event-time*, it can guarantee the *exact-once* execution but since a lot of operations need to wait the watermark, so *higher latency* is caused.
For an application using *process-time*, no need to wait watermark, so latency is limited, but when it comes to recovery, process-time will cause inconsistent replay.

Flink also offers a trade-off, *ingestion-time*(i.e. the time when an event enters Flink). With ingestion-time, trade-off can be attained between latency and accurate results

#### Streaming Processing State

State is essential for most streaming applications, and Flink also provides flexible custom state management. In Flink, state is made explicit and is incorporated in API by providing:

1. explicit local variables registering
2. explicitly declaring partitioned k-v state and associated operations.

#### Stream Windows

For Stream data, windows operations is an important concept. Generally, windows operation consists of:

1. *assigner* : The assigner is responsible for assigning each record to logical windows.
2. optional *trigger*: when the operation associated with the window definition is performed.
3. optional *evictor*: which records to retain within each window.

#### Asynchronous Stream Iterations

Feedback is also need to be considered as operator and be snapshotted in iterations operations.

### Batch Analytics on Top of Dataflows.

Although batch process can be considered as special case of stream process, Flink still offers specific support for batch process for at least two reasons: simplified API and batch-specific optimization.

For batch process, some existed optimization can not be used for some state are not transparent for out-system, and Flink offers some its own way to do optimitations. 