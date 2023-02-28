# The Design of a Practical System for Fault-Tolerant Virtual Machines

## Introduction

### What is the main topic of this paper ?

Basically, the writers design a system for fault-tolerant based on **back-up**, with poor need of network rate and easy to use, and furthermore has been applied into commercial usage. 

However, that's means *back-up* need to hold almost the same state with the main server to replace the failed machine as soon as possible. And there is some existed ways to do that:

1. *State transfer*: Transfer all data including CPU, I/O to the back-up server, which needs high bandwidth demands.

   Useful in multi-core situations.

2. *state machine approach*:  Use instructions sync in different state machines. In this case, we can use less cost to keep back-up consistent with the main machine. However, since machines are not always deterministic for all instructions, a little extra  information is needed. For physical processors, this is very difficulties since the high frequencies.

   However, if the main server runs on a hypervisor, these non-deterministic instructions can be copied exactly.
   
   The main topic in this approach may be:
   
   1. What kind of state do we need to transfer ? 
   
      This topic basically focus on in which level should we replicate, for example, application level or in this paper system level. It's natural to find the trade-off between cost and robustness. 
   
   2. What degree of sync do we need to maintain between primary and backup?
   
   3. How to implement cut-over in primary or backup failing ?
   
   4. How to deal with the anomalies in cut-over ?
   
   5. How to create new replicas in a cheap way ?

And this paper only focus on fail-stop failure, which can be detected **before** the failing server causes a big trouble.

## Main Topic

Basically, there are three main ideas in this paper, *Commands transfer*, *prevent data lost*, and *detect failure* .

### How dose the system keep commands sync ?

In this system, commands are transferred through **logging channel**, and we need pay the most attention to the **non-deterministic** commands. 
Hence, for replaying all states changing, we need to do three things:

1. Capture all deterministic commands and all non-deterministic effects;
2. Apply all things captured above into back-up VM
3. Affect only very little in normal running.

And, here non-deterministic effects includes **non-deterministic events** (network timeout) and non-deterministic operations(read the clock cycle counter) and undefined behaviors. So, the *most challenging part may be capture the non-deterministic effects*.

VMware also uses log to do this. For deterministic operations, log records only the commands, and for non-deterministic operations, log will records a lot more information to guarantee replay the same state changing in back-up.

Some has come up with a epoch idea to reduce cost by frequently interruption, however, current system is enough to use.

### How does the system prevent date from being lost

After the primary capturing the information and logging, the system will send the logs with network called **logging channel.**

Actually, after a failover, the back-up machine is likely to execute in different way with the primary machine since there could be many non-deterministic operation in the execution. However, if only the output can satisfied the requirement and be invisible to clients, it's just feasible.

To satisfy the output requirement, *even the external output can be delayed* to ensure the back-up VM indeed has all prior conditions. 

If a primary VM crashes right after the execution of an output operation, we still need the back-up machine replay it up to next of the output, since there may be inconsistency if not doing that. So, *FT protocol* just give an special log to output, which **must get ack** from back-up before doing output. Additionally,  this process can run  asynchronously instead of waiting the ack.

And the system can *not guarantee to run exactly once*, since back-up never know if primary will crash before the output and after the output. However, it doesn't matter in this case, since the network infrastructure and OS are born to deal with replicated and missed packets, and in our cases, they are almost the same condition from perspective of OS network part. 

### How does the system detect node failure, including primary and secondary.

- Backup VM fails

  Just do not record and send to the Backup VM

- Primary VM fails

  Need continue replaying commands in the logs , and need to do extra work such as broadcasting the MAC

   address to do normal connection.

To detect the failure, there could be two ways:

1. heartbeat 
2. FT Protocol monitors all logging traffic 

However, there still could be a split-brain problem, that's to say,  in some cases ,we can not figure out whether the timeout is caused by VM crash or just network, so additional lock should be used in remote shared storage to ensure the exclusive invocation, which is basically no extra overhead.

Also, after a failure of one VM, the system will invoke another VM as soon as possible to be the new BackUP.

## Implementation

### Start a backup VM in the same state as the primary

Based on *VMotion*, the *FT VMotion* can clone a VM through network less than one second.
And, since the shared storage is used in the system, all machines in the same cluster can be used to be backup. Also, the cluster manager will maintain a service to determine which is the best VM based on the network and other resources usage.

### Manage the logging channel

All the commands executed by the primary VM will first be added into a large log buffer in hypervisor, and then be transmitted through log channel to the buffer in Backup VM, just like the producer-consumer model.

In the implementation, we need to make sure that the buffer is rarely full for two reasons:

1. If the buffer is full, the primary VM should wait until one log can be added into the buffer, so the interaction between primary VM and the client will be effected.
2. If the buffer is too heavy, the time for backup VM going alive may be too long since the backup need to replay all acknowledged but not executed logs.

Basically, the method is to use FT protocol and hypervisor to be the manager so that the computing resources can be dynamically adjusted in primary and backup VM to ensure the reasonable buffer usage   

### Operation FT VMs

Almost all control operations should be synced to the backup machine, such as the CPU share and power off.

The only operations can be done independently on the primary and backup VM is *VMotion*. For a backup VM, the VMotion requires all IOs being done before some point during VMotion, and it is difficulty since the backup should do as the primary VM does. The FT VMotion do an additional operation that the backup send massage to the primary VM for a temporary quiesce.

### Disk IOs

1. There may be some rare cases of disk IO races, but we do not use lock or so to ensure the mutual execution for better parallel efficiency 
2. However we still need some mechanism to avoid the race. Page table protection is a solution but with too high overhead. And set a bounce buffer could be an alternative, which can be a interval when race happens. All read and write operations must be carried out in bounce buffer first and then carried out in the real disk.
3. When it comes to failure during Disk IO, since we do not have any ack for disk IO, so the backup will never know whether a disk IO succeed or failed with issues. However, we can just repeat the DISK IO no matter it has executed or not.

The above mechanism of course add additional overhead to IO, however, it is not obvious and acceptable.

### Network IO

For normal VM, the hypervisor will do network IO asynchronously, which will increase the non-determinism.  

We need to reduce the asynchronous optimization without obvious loss to network, typically, the hypervisor uses *trap* mechanism to do network IO, so the following methods can be used to reduce network overhead:

1. clustering optimizations will reduce the  number of trap.
2. reducing the delay for transmitted packets in hypervisors.

## Design alternatives

### non-shared memory

shared memory can decouple the persistent storage form the design, however, there is some cases that non-shared memory could be a better choice, such as higher redundancy or shard storage itself is impossible or too expensive.

For a shared memory, the backup do not read or write the disk, and the disk operations itself can be considered as an interaction with the external world, so it is also need the ack from backup to carry out write operations.

However, for a non-shared memory, the backup need to do write operations itself, and write can be considered as an internal operations. And, if we use non-shared memory, additional methods should be added for split-brain problems such as a third server or so.

### backup read

Typically, backup consider read as input, so the backup just obtain read data form the logging channels as other input information.

If we allow backup to read disk, it is obvious that the logging channel will feel better.

However there may some other problems:

1. The read in backup must wait until the same operations in primary finished, this will slow down the execution of the system.
2. Do not friendly for failure no matter primary failure or backup failure. For former failure, the failure related message must be transmitted through logging channel to the backup as before, and for the later failure, the backup should retry until it succeeds. During this process, the related operations in the primary will be delayed.
3. Specifically for shared-memory, the write and read operations in both primary and backup should be ordered strictly, or the determinism will be damaged.

