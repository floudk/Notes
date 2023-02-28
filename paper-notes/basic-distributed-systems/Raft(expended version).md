# In Search of an Understandable Consensus Algorithm

## Introduction and Background

### What is raft

Raft is a **consensus Algorithm**, being almost identical to Paxos, Raft is much more understandable, which is very important in teaching and industry area.

### What is *Consensus Algorithm* meaning

Consensus algorithm help a cluster consisting of many machines survive the failures of part of machines, which plays an important role in large system building.

### Why we need Raft instead of other consensus algorithms

Basically speaking, before raft, consensus algorithms are dominated by Paxos or Paxos-Based algorithm, which is a disaster for educating and practical using in industry environment.

## Mechanism

### Why we need **Consensus Algorithm**

We need consensus Algorithm in the **replicated state model**. 
Basically speaking, in distributed system, we need all server have same state, so we should keep all input order as the same contents and order. Here we introduced **consensus model**. 
Consensus model will receive input commands and communicate with other servers to make sure all servers will have same command log.

An eligible consensus model must own the following 4 features:

1. Keep safety. Never return wrong answers in all non-Byzantine conditions.
2. Fully functional.  As long as the majority machines keep alive the model can keep going.
3. Do not depend on timing to ensure the consistency
4. A minority of slow servers do not effect the whole system.

### Why not **Paxos**

1. hard to understand in education area.
2. No uniform implement or standards.
3. Do not consider physical usage in design, so every time coders need to begin with Paxos and end up with something different.
4. Not friendly to industry application, since decouple multiple choices into single-decree ones will actually make things more complicated 

### Raft Algorithms

For better decouple, Raft can be basically decoupled as three parts: **Leader election**, **Log replicated**,and **safety details**. 

> All schemes are to ensure that machines in the same cluster can execute the same commands in exact same order, thereby ensuring the consistency of all state machines.

In raft algorithms, time can be split into **term**. During one term, an unchanging leader will be elected and be responsible for maintaining log consistency, Each log item will include one log index and the term index.

Therefore, we can decompose the problem for discussion:

#### Leader election

There are three types for servers in Raft algorithms: **Leader**, **Follower**,and **Candidate**. 

At the beginning of each term or current leader is unavailable for some reasons, parts of the servers will transfer to *candidate* and send **RequestVote PRC** to all other servers. For other servers, they will reply the first received RequestVote PRC as a vote, and the first candidate who receive the majority servers' votes will be the leader, at the same time, all other servers in the same cluster will transfer to followers.

If there is no candidate wins the election, the next term and next election will be invoked immediately. Additionally, Raft uses randomized election timeouts to ensure the above useless election is extremely rare.

#### Log replicated

After a leader elected, it's the leader's responsibility to keep consistency in log replication process.

Basically speaking, all order logs can be split into two types: **uncommitted** and **committed** logs. All servers will execute orders after the logs being *committed *. Hence,  the core of the problem is how to commit logs.

During each term, the clients will only communicate with the leader (if a client contacts a follower, the follower redirects it to the leader). After receiving a command from client or so, the leader will create one entry including log index and term index, and then send *AppendEntries PRC* to all followers. Once a majority of machines in the cluster reply successfully replicated, the entry will be considered as *committed*, and be executed by all machines.
At the same time, the *AppendEntries PRC* will check whether the last entry in followers' logs is identical to the leader's, if checks fail, the leader will force the followers' logs to duplicate its own.

In this process, an interesting case is that the leader fails for some reasons, which will be solved in next part- safety details. 


#### safety details

1. Leader election

   For simple implementation, Raft only allows single-direction log  transmit, hence, we need to ensure the leader always owns the most up-to-date logs.

   In order to do so, Raft designs a mechanism in leader election. When communicating with other servers, the candidate will send its log information besides the RequestVote PRC, all receivers will contrast the candidate's latest log index with local latest log index, and only vote in candidate do not out of date.

   Since a candidate need to be admitted by majority of the machines in cluster, so after successfully elected, the leader will be ensured to be the most up-to-date.

2. Committing entries from previous terms.

   There is a troublesome problem in consensus algorithms: what if leader fails, what should new leader do to non-committed but already in majority servers log after elected.

   Raft prohibits the submission of logs of past terms by counting in the majority, and non-committed past logs can only be committed with committing logs in current term.

3. Safety proof.

   **Leader Completeness Property**: For a given log committed in term M, any term N > M will hold the log.

   **State Machine Property**: For a given order applied in a machine, No other machines will apply other order in same index.

   The leader completeness property is the base of State machine Property. And the state machine property ensures that all servers will apply commands in the same order.

#### Follower Crashes

Follower Crashes are much simpler than leader, just retry send RPC indefinitely until receiving responses.  And the same RPC has no effect on current system.

#### Timing and Availability

Raft consensus do not need time to guarantee the consistency and availability, once if the time limitation is satisfied: 

$T_{broadcast}<<T_{election\_timeout}<<T_{Average\ crash}$

### How raft change cluster configuration without offline?

Raft uses a kind of  intermediate state call *joint consensus* to  solve this.

After configuration changing, all things run as above. Once a new configuration request is made, the leader will create a log called $Configuration_{Joint}$, and try to repeat it like other log. All servers will read the new configuration as soon as it is repeated. During the transition period, servers in a cluster can run in $Configuration_{old}$ or $Configuration_{joint}$ without any conflicts.
After $Configuration_{Joint}$ committed, the current leader must be server in $Configuration_{Joint}$, and then we can transmit to $Configuration_{New}$ as the above steps.

Also, there should be three details to be noticed.

1. Some new servers may not own any logs, so it could use a long to time attain consistency as the older nodes.

   Raft set another intermediate time to do so, in this period, the new leader will act as audience, that's to say, can only receive logs as followers but owns on rights to vote.

2. A leader may manage a cluster which does not include him.  It doesn't matter for this. 

3. Removed servers may affect the running of whole system. For unused nodes receiving no heartbeats(since the leader will use the new configuration), they will invoke new election, and do it again and again. So Raft will answer no RequestVote for believing there indeed a leader.

### Log Compaction

As state machines running, the log will become larger and larger. However we can not running in unbounded memory machines. So we have to devise a method to reduce the sizes.

**Snapshot** is a good method, storing state machine's current states and necessary metadata. Also some incremental approaches such as *log cleaning and log-structed merge trees* are also feasible and all share the same interfaces.

In snapshot, raft no longer insists on the leader's decision-making power. All servers will independently make its own's snapshot. The reason why do this is that not only all infos needed by snapshot existed in servers already which is guaranteed to be consistency, but also in leaders' transmitting the snapshot will couple with the new coming requests, which deepens the complicity of system.

In this case, basically speaking, hardly will snapshot transmit through network. However, if exactly the last log in leader is in snapshot, the leader will send snapshot with *InstallSnapshot* RPC. When receiving a *InstallSnapshot RPC*, the follower will choice how to deal with it. If the received RPC including logs not existed in local replication, the follower will discard local states, if local state including some extra logs, it will receive the leader's log while keeping the extra parts.

### How does raft interact with clients？

 For interaction with clients, two problems need extra attention. 

First, *how clients find the leader*? Basically, the client will connect with a node randomly, if the receiver is not a leader, then information related to leader will be sent to the client. Furthermore, if during interaction, the leader crashes, the client will set a timeout and connect with a random node as well.

Second, how  to guarantee *exactly-once* ? If a leader crashes right after committing a log but before responding to the client, the client will send the request to a new leader, causing a command being executed twice. Hence, raft assigns unique serial numbers to every command, so that servers can track it and avoid being executed twice. For read-only command, leader must reply after a heartbeat to ensure no stale-data is sent.



---

### Ref

1. [visualization](http://thesecretlivesofdata.com/raft/)
2. [Details Blog in Chinese](https://zhuanlan.zhihu.com/p/314927071)

