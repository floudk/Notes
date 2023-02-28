## Motivation

*SPE*(Streaming Processing Engine) is increasingly important in these days, however, SPEs suffer lack of elasticity in some degree. 
SPEs are frequently used in dealing with streaming data, which varies as time and region including predictable and unpredictable sakes. But roughly all SPEs are not able to do scaling on the fly, and they need to stop and restart with new configuration. This procedure is really an issue for many latency sensitive applications.

## Introduction

Motivated by the above problems, this paper proposes a protocol to adjust resources on-the-fly while ensuring exactly-once processing semantics.

Further, the paper implements this idea in Flink, enabling:

1. stateful operators migration
2. new operators introduction
3. existed operators changing based on external triggers.

## Background

### Data Stream Processing

SPE are not a much novel idea, and it can date back to almost one decade ago, but it has only recently begun to develop rapidly with the advent of advanced cloud computing infrastructure and greater demand for real-time data processing.
New SPEs adopt one of two processing models

1. *micro-batching*: split the unbounded data into a series of bounded chunks of data, in which the size of chunk determines latency and throughput.
2. *tuple-at-a-time*: some kind of real streaming process engine, dealing with one tuple at one time. However, in practice, this model still uses batching method to ensure high throughput.

A natural way to modeling data flows in SPEs is DAG. All nodes can be split into three types:

- source: input
- processing: interval nodes, can be either *stateful* and *stateless*.
- sink: output

### Apache Flink

Flink uses the following models to perform concurrent computations 

> *PACT programming model*: a generalization of the MapReduce programming model and uses second order functions to perform concurrent computations on large (Petabytes) data sets in parallel. PACT generalizes a couple of MapReduce's concepts:
>
> - Second-order Functions: PACT provides more second-order functions. 
>
>   > second-order functions:  takes one or more functions as arguments and returns a function as its result.
>
> - Program structure: PACT allows the composition of arbitrary acyclic data flow graphs. In contract, MapReduce programs have a static structure (Map → Reduce).
>
> - Data Model: PACT's data model are records of arbitrary many fields of arbitrary types. MapReduce's KeyValue-Pairs can be considered as records with two fields.

Flink uses buffering and pipelining to maintain high throughput and low latency, and an operator sends its buffer downstream when it is full or attain a predefined time-out.

**Back-pressure** back-pressure occurs is Flink when an operator receives more data than it can actually handle, and it is usually due to a temporary spike, stalls, or network latency.  

### Checkpoint in Flink

Checkpoint lay the foundation for the savepoint mechanism, and it enable specifying message delivery guarantees, which can be processed exactly-once, at-least-once, or at-most-once.

Through checkpoint, the process can restart with adding or adjusting operators. For tasks with small states, the checkpoints have only negligible impact on overall system performance, however, for large state tasks, checkpointing may have a giant difference on system performance.

Also, checkpoint mechanism needs replayable data source.

## Protocol Description

### system model

- *Tuples*: r1,r2,... $r_j=(A,t)$ consists of attributes A and a timestamp t.
- *UDF*: user defined function: $f(r_j,S)$, S is the current state of the operator.
- *DAG*: Flink uses DAG to represent a processing task, the DAG edge is kind of communication channels between nodes or operators, and can exchange data in three ways:
  1. Forward a record to a single instance of downstream operator.
  2. Broadcast a record to all instances of downstream operator.
  3. Send a record to a set of instances of downstream operator grouped by partitioning function.
- *Marker*: $m_i$ can be a watermark, triggering the i-th checkpoint for a job. After get marker from all instances, an operator will reacts by snapshotting and forward the maker.

### Migration Protocol Description

For dealing with possible hardware failure, we do need some way to migrate all slots in a node to other node, and this protocol is just for this.

The system will create a special modification marker containing all necessary information to perform the migration and ingests it into the DAG. All server in the DAG will receive the marker and decide whether it needs to react on that migration marker.

Notice that, the migration will affect three kind of operators: up-stream operator, down-stream operator and itself.

- *Upstream operator*: buffer their records during the migration duration with FIFO order.
- *Migrating operator*: store their state in persistent storage in order to consistently resume processing tuples once restarted on a different worker.
- *Downstream operator*: rewire their inbound communication channels  

### Modification Protocols Description

For a running job, we may need introduce new operator instances or replace the operator function in a running job.

The SPE starts each modification similarly to a migration by ingesting the modification marker at the source operators.  

#### New operator introduction

- *upstream operators*: buffering their outgoing records, and then broadcasting the upcoming location of the newly introduced operator to all downstream operators  
- *New operator*: all new operator instances will be started  right after all upstream operators have successfully acknowledged buffering  

The UDFs compiled code need to be sent to the target nodes first.

#### Changing the Operator Function  

Upon receiving the checkpoint marker for that modification, each instance of the target operator instantiates the new UDF and continues processing records.

## System Architecture  

### Vanilla Components Overview

In origin model, the control messages are not part of the actual data flow and are realized as RPC via *actor model*.

> actor model: The actor model represents a system, in which all entities, namely actors, run concurrently
> and solely communicate via message passing.  

Flink handles the data exchange among parallel instances of two operators in a produce-consumer relation through an internal protocol based on top of TCP.

Flink leverages either the Job Manager or a third-party storage system to persistently store the checkpoint data.  

<img src="D:\EndNote\paperNotes\Research\Flink\imgs\Vanilla Flink Control Msg Model.png" alt="image-20220331183210651" style="zoom:50%;" />

### Changes on coordinator side

In Flink, there is a Checkpoint Coordinator to manage all operations related to checkpoint.

Similarly, this paper introduce a *Modification Coordinator* to handle and validate all modification related operations.

Firstly, the *modification coordinator* will check the validation of modification command. If success, a modification-specific trigger message will be prepared and ingested to the data flow at source.  The coordinator keeps track of modification life-cycle in all vertices.

Each operator will react or ignore the message, but all operator need acknowledge the reception to the *Modification Coordinator*. Depending on whether all vertices successfully acknowledge the trigger message as well as all subsequent, modification-specific state changes, the coordinator eventually marks a modification as completed or as failed.  

> State: Normally a Flink task starts in the **Created-state**, gets **Scheduled** and **Deployed** by the Job Manager, processes all its input in the **Running** and eventually finishes by entering the **Finished** state. 
>
> A task may fail for various reasons and subsequently enters the **Failed** state.     
>
> Finally, if the SPE cancels a task due to an external reason, it enters the **Canceling** and **Canceled** state consecutively.  

The paper introduces two new states: **Pausing** and **Paused** state, which the task enters when the SPEs triggers a migration. Notice for stateful operators, the tasks should submit these state to checkpoint and store it in a persistent storage system.

### Changes on the Worker Side  

Normally, for task manager, it should do four things:

1. deserialize incoming records
2. process records
3. serialize output records
4. place them in outgoing queues.

The Job Manager determines the location of the up- and downstream operators in the job’s initialization phase, which may be in memory-queue(producer and consumer are in the same task manager) or network(producer and consumer are in different task manager).

The up-stream operator should buffer all output and do not block more upper operators until the migration is finished. The buffer can be in-memory or in-disk with FIFO order.

### Query Plan Modifications

The difficult part is how to do the above protocol without hurting data integrity.

For each single operator instance to modify, its up- and downstream operators should also be considered, so the modification message contains complete instructions for all operators.  

#### Upstream Operator

Like snapshot, operator also need to do *align* for modification message maker with buffering some commands. However, these buffer are roughly independent to Flink internally-managed state, it should be handled separately.

All up-stream operators should perform a custom alignment procedure on the sending side.

#### Target and Downstream Operator

downstream instances receive the new location in the last record and then update the connection to that new upstream instance accordingly  

## Protocol implementation

Basically, these operator migration, modification, and updating all have similar mechanism, which may be very ambiguous in description and might be clear in code.

 First of all, for these modifications, collecting and reassigning state are all relaying on *checkpoint mechanism*. 

Typically, the operator migration can be split in 4 steps:

1. *Resource allocation*: Upon receiving a migration request, the MC will retrieve all operators on that specific TaskManager, and try to allocate new slots to each migrating operator.
2. *Message generation*: The MC(Modification Coordinator) will creates the **trigger messages** so that the exact operators (including up and down stream operators ) can be triggered by subsequent checkpoint maker to do the migration.
3. *State Collection*: When the maker come to the Migration operators, it will collect its state and send it to the persistent storage. At the same time, the operator will transmit its state to be *Pausing*. Then, it sends the new location to all downstream operators. After the above operations, The to-be-modified operator transmit to *Paused* state, and prepare to release the resources. 
4. *Operator Restart*: Use the checkpoint mechanism to reassign the state to new operator location and all become normal soon.

For introducing new operator and updating UDF in current operator, the steps are basically identical, but system need to transmit the compiled code firstly.

## Evaluation

The results show that migrating operators for *jobs with small state* is as fast as using Flink’s savepoint mechanism. Migrating operators of *a job with 15 GB of state* even outperforms the savepoint mechanism by a factor of 2.3  