### Motivation

Flink currently do not support dynamic scaling mechanism, causing either workload may exceed the computation resources or some computation is wasted.

This paper propose an architecture that enables the automatic scaling of Flink jobs.

### Introduction

Although Flink do not support change parallelism during runtime. It is still possible to take a savepoint and then restart the job with a different parallelism from the snapshot. 
But this is an very expensive operations, besides the ABS snapshot operations. restore from the savepoint will take a considerable amount of time, and Flink need to catch up with the continuing unprocessed jobs need additional delays. *So it is important to do the scaling at the right time to be worthy.* Container orchestrators, such as *Kubernetes*, can help us to automate the container to attain the aim.

Based on Kubernetes relevant tools, this paper proposed a simple scaling policy according to the operator idleness and changes of the input. Additional, this paper analyze the scaling process. 

### Related work

A classical scaling procedure consists of two phases:

1. When to do the scaling ?
2. How to do the scaling ?

And most work are focused on the former one. T. Lorido-Botran  had concluded related methods, and categorized the techniques into five categories:

- threshold-based rules 
- Reinforcement Learning
- Queuing Theory
- Control theory
- Time series analysis based approaches.

**DS2**  proposed a lightweight instrumentation to monitor streaming application at the operator level.

**PASCAL** is a proactive autoscaling architecture, use a machine model to predict workload and performance.

Ghanbari et. al.  proposed a model based on cost.

### System architecture

#### Kubernetes operator

Flink applications can be executed in different ways including Kubernetes. And Flink can run on Kubernetes in both standalone and native mode. In standalone mode, Flink have no idea of the underlying API and only know what Kubernetes offers to him, and In native mode, Flink can interacts with the Kubernetes API server.

This paper uses a standalone mode combined with Kubernetes' operator pattern to manage Flink resources. And google open-source API offers special support for that.

#### Scale subresource

The scaling process starts with this step.

As the scale subresource's replica specification changes, the operator first request a savepoint and then delete the existing cluster. After the cluster components are ready, It resubmits the job with the appropriate parallelism, starting from the latest savepoint.

#### Custom metrics pipeline

This paper uses Prometheus metrics to be the metrics of Flink jobs, and uses an adapter to access Prometheus metrics through the Kubernetes metrics API.

#### Horizontal Pod Autoscaler

Horizontal Pod Autoscaler  is Kubernetes built-in resource, which can control the scaling of Pods in a replication controller.

### Scaling Policy

Inspired by this [blog](https://www.ververica.com/blog/autoscaling-apache-flink-with-ververica-platform-autopilot). This paper assumes the Flink job reads data from Kafka and emits the results to another Kafka.

The scaling policy uses two metrics:

1. the relative changes in the lag of the input records
2. the proportion of time that the tasks spend performing useful computations

#### Relative lag changes
Kafka use *consumer lag* to keep track of how much consumer is behind producer.
Although Flink uses a different mechanism to do that, it can also get the info from Kafka. 


### The effects of scaling on performance
In this section, we analyze *how the size of the state affects the downtime during the scaling operation*.  

Since the node will do snapshot after it has received all barriers from inputs, so *when operators are under different loads*, this can significantly delay the snapshotting of certain operators' states. That's so called **align checkpoint**.

Flink currently support **unaligned checkpoint** now. (but no unaligned savepoint.)

Basically, savepoint time is roughly linear to the state size, and the jobs parallelism have little effect.
Also, the I/O rate of the storage system also can be a limit.

#### Effects of state size on scaling downtime

The majority of the downtime during a scaling operation is due to t*he loading of state from the persistent storage*.  

The file system is an important factor to effect to downtime, especially the infrastructure's creation, deletion or so. Also, to get the Flink image, these part of time is also unpredictable. In order to explore further, we assume that the infrastructure is always sufficient.

We have not found a correlation between the parallelism of the restarted job and the maximum or the average load time of its subtasks  
