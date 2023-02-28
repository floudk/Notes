## Chain Replication for Supporting High Throughput and Availability

` Chain Replication` is an approach to coordinating clusters of fail-stop storage servers, designed for supporting high throughput and availability while keeping consistency as much as possible.

### background

Classic Storage system includes file system and database system, and it can be said that file system is kind of subset of DB. However, in this paper, we focus systems between File system and DB, since in some cases, DB is too big to be implemented while File system is not complicated enough to support project, which can be called *storage service*(etc. GFS, Hadoop or so) in this paper.

In large-scale storage system, availability and high-throughput is always incompatible with strong consistency. However, `chain replication` is proposed to *attain three aims simultaneously*.

And this paper focus on the *interface*, *implementation*, *contrast*, *simulation* and *Related work*.

### Interfaces

Let the server, instead of the clients, guarantee respond to every request from client is never a smart method. And for a client-guarantee response system, the frequency and duration of server outages must be limited, hence, `chain replication` is proposed to reduce recover time while not sacrificing consistency and blocking normal operations.

In this paper, storage system has state and operations:

- State:

  state consists of two operations array, 

  - *Hist*: Requests already applied (do not need contain *query* operations)
  - *Pending*: Requests to be applied soon.

- Transactions

  1. T1: Request arrives and be inserted into Pending
  2. T2: Request is ignored and deleted from Pending
  3. T3: Request processing, 
     for *query* just return reply; for *update* operations, do update and insert into Hist.

### Protocol

By meaning *fail-stop*, it means that:

1. A server will stop when confronting with errors instead of transferring into Error state
2. We can detect the stop situation.

For a cluster running chain replication protocol, it can tolerate at most t-1 failure, and it can handle 2 kind of things:

- **Query(read)** Processing

  All such requests will be directed to the tail server, and the tail will response according to local replica.

- **Update(write)** Processing

  All such requests will be directed to the head server, and the head server will execute the command and forward the state change to the successors. That's to say, all computations will be only carried in head server, making non-deterministic commands can be feasible.

This mechanism will cause two results obviously:

1. All reply from the system will only be generated in the tail server.
2. Read(Query) are much cheaper than Write(update) in this system.

At the same time, we should ensure the consistency of the state space of the system, which needs to be robust in all cases.

#### server failures

First of all, we need a **master** node, which will take the responsibilities: **failure detection** and **communication management**. In this paper, we assume the master will never encounter any failure (in practical, we can use Paxos or so to attain the aim.)
Based on the master server, we can solve the failure of three servers.

##### Failure of Head

The failure of head have no effect on $Hist_{objID}$ ,  and may cause discord a request in $pending_{objID}$, and this behavior is legal, which means it is hard to tell the difference between the failure of the head and normal run.
What the system need is just let master know this and inform relevant objects. 

##### Failure of Tail

The failure of tail will cause $Hist^{s-}$ be the new $Hist$, and $Hist^{s-}$ owns more items than origin $Hist$, which should be deleted from $pending$, since the new tail has already received the items.
Also, the state change is legal.

##### Failure of Other

The failure of internal server will be more troublesome, since the failure of $S$ will lose some log, which appear in $pending$ and $Hist^{S-}$. That will cause in-consistency in state space. Hence, additional measure should be implemented to keep the consistency.
$Sent_i$ is introduced to handle this. $Sent_i$ means some logs processed local yet and forward to the next but may not attain the tail server. The tail need send back an ack for each log it received from the chain, and the ack will do reverse chain propagation.
When encountering the failure or interval server, the master will do the following steps in  order:

1. Tell the $S+$ server the new role and get the newest index in $Hist^{S+}$
2. Tell the $S-$ server the new role and the new latter's $Hist$ condition
3. The $S-$ will send what $S+$ need to it.

During this, $S-$ will be 'blocked' and do not process any new requests.

#### Chain Extending

Besides Chain failure, we also want extend the chain to keep the number of total servers in case of server failures.
Since we have a chain, we can do a little trick to do it much easily. We put the new server behind origin Tail server, and make it the new Tail.
In terms of states, the origin tail should copy $Hist$ to the new Tail. During this process, the origin tail can work like an interval server, processing request and append it to $Sent$, and for the Query directing to the origin tail, the origin tail can just discard them, or send forward to the next. After the copy, the master should never consider any origin tail.

### Chain replication & Primary/Backup Protocols

For classical Primary/backup protocols, a reply is made with 4 steps:

1. Primary receive a request and do computation
2. Send info to followers
3. wait ack from all non-faulty followers
4. send reply

In a sense, chain replication is kind of extended Primary/backup protocol. It distributes the responsibilities of the primary to UPDATE and QUERY. 

In this case, query can be much cheaper, since it will be only processed by tail and no not include any coordination or other computation tasks. However, the chain replication theoretically insert UPDATE delay.

Despite of above difference, we care much on the server failure. *Will client face a transit outage in server failure?*, *If no transit outage occur, how much will the message delay?*. Here, transit outage means a client have to stop and wait the reconfiguration of clusters.

By the way, the detection of a failure is of course expensive, however, they are identical in 2 cases, so we just ignore it, and the following discussion is based on the assumption that the failure has already been detected.

#### Primary backup protocol

For Primary?backup protocol, the server failure can be split into 2 kinds: primary failure and backup failure.

1. In primary failure, client must stop work and wait at least 5 message time.
2. In backup failure, if one update request in just on processing, a one message outage will be needed.

#### Chain replication prorocol

1. Head Failure: Query works well, and Update will be unavailable for 2 messages.
2. Middle Failure: Query works well, and Update can still work, but delay for 4-message-time will be needed.
3. Tail Failure: 2-message-time will be needed to make Query and Update work.

### Simulation

Through simulation, The paper contrast the 4 cases

1. Chain Replication
2. P/B
3. Weak Chain Replication : Query request go randomly instead of only tail
4. Week P/B: Query request go randomly instead of only primary.

#### One chain, no failure

In this case, two weak method is almost identical, Basically: $CR > Week > P/B$

The weak methods only perform better in the case where Query requests occupy more than 85%. It's of course natural to get the conclusion, since random dispatch the Query means the head also has probability to get Query and affect the Update computation.

#### Multi chain, no failure

All methods perform almost identically in this case.

#### Failures on Throughput

For a server failure, the system  do three steps : detect-> delete -> restart a server.
To deal with failures, the throughput of a whole system will change to respond to the failure.
Generally speaking, for multi-chain, the failure will cause 2 stages:

1. Transit outage for about 10 seconds. During this period, the system will be blocked until the master detect the failure and start related work.

2. For query request, after detection, the mater delete the failed server, and add a new server as the tail to all chain including the failed server, so the work load can never as high as before, since a common tail need to deal with all query from multiple chain.

   For update request, after detection, the throughput will increase higher than before, since a server failure causes some chain have shorter chain length, so it can process much faster.

### Large Scale Replication of Critical Data

As mentioned above, when doing recovery, it is not reasonable to insert a server as tail of multiple-chain, not because it increase the work load of this server, but also be especially awful in large scale replication, since the tail must get the copy of all existed contents.

Because the replay process, some volume have to be unavailable in large scale replication, however, the parallel recover can reduce the unavailable time.



   

