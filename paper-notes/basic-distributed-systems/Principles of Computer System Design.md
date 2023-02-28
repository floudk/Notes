### 9.1.5 Before-or-After Atomicity: Coordinating Concurrent Threads  

For any concurrent programs, there must be coordination requirements. There could be at least two kinds of methods:

1. *sequence coordination*: It is an explicit constraint which defines which action should be first and which should be the second one. In this case, all actions are instructed orders by commands.

2. *before-or-after atomicity*: It is an implicit constraint method, users do not need know any knowledge of which action should be first in the system.

   > Concurrent actions have the before-or-after property if their effect from the point of view of their invokers is the same as if the actions occurred either completely before or completely after one another  

### 9.1.6 Correctness and Serialization  

For a concurrent system, we can see there are many instruction orders to do the work, but some are correct, others are wrong. So how can we define a correct order no matter which kind of task are executed?

For *before-or-after* atomicity, we can define that *if every result is guaranteed to be one that could have been obtained by some purely serial application of some actions*, then the order is correct. Namely ,that means **serializability**. Moreover, we can even do not care about the intermediate state of such system if the intermediate states are guaranteed to be invisible for users.

However, the aim of the concurrency is get high performance as possible as we can. In all possible orders, there must some orders are not such good, but how to choose the best order is an advanced and difficult topic.

Of course, we can demand higher atomicity than before-or-after atomicity model, since it indeed do not satisfy some situation, for example, the operation with timestamps. 

### 9.5.2 Simple Locking  

Simple locking is just simple locking. No matter read or write, simple locking mechanism will try to get all locks on relevant data.

Of course, this mechanism always can not acquire the highest performance, and miss some opportunities of concurrent executions.

### 9.5.3 Two-Phase Locking  

Basically, the two-phase locking is that:

1. Get all locks needed when beginning the transactions.
2. Release locks if data has only read before and do not need read in the future, namely, never release write lock during a transactions.

Still, this mechanism although can do better than simple locking, but also miss some concurrency.

Another interesting question could be do we need to keep locks non-volatile during crash and recovery based on locks and logs system. Generally, we do not need to do that. Since when recovering, the system will do *undo* and all data object involved in these procedures should first be guaranteed to share no overlap in lock set, otherwise, the concurrent transactions will never do in this system.

### 9.6.3 Multiple-Site Atomicity: Distributed Two-Phase Commit  

When considering multiple-site atomicity, the situation can be much more complicated since messages in **a best-effort network** can be lost, delayed, or duplicated.

In order to get a reliable protocol, we need a hierarchical two-phase commit.

For transaction manager(coordinator), it implement a high-level two-phase protocol. And for workers, they use a low-level 2-phase commit, which is actually three phase: ready to commit, commit tentatively, and committed.![image-20220518145201985](imgs\Distrubuted 2-phase protocol.png)
