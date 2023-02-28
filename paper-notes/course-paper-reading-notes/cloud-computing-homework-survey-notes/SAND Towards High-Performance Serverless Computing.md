### Introduction

Currently serverless platform will *run functions in isolated containers and not do any interactions among functions*, causing high startup delay and resource waste.

So **SAND** serverless system is proposed here, providing lower latency, better resource efficiency and more elasticity with 2 key techniques:

1. **application-level sandboxing**  

   1. isolation between different  applications
   2. isolations between functions in the same application

2. **a hierarchical message bus** 

   To use the locality in the same application.

   Let the functions share the same information, and use a global backup to ensure reliability. and used the same method on storage 

The results shows this implementation is much better than current systems with containers.

FaaS model offers ephemeral execution environment for a request, shifting the responsibility for dynamically managing resources to the service provider. However, currently, serverless can only support simple functions since the high level overhead.

According to the writer, the reason for the high overhead can be split into 2 parts

1. *separate container instance* : This will cause well-known *cold start* problem, and some system keep a container being *warm* for a certain time, which indeed mitigates the cold start problem, but also make resource be inefficiency.
2. The existing platform does not consider the interaction between functions, and adopts the same mechanism for internal functions and external functions, which leads to some redundant behaviors and increases the cost.

### Background

Containers-based system is common currently.

To reduce overhead of cold start, common methods are:

*keep them warm for some time, and even pre-warm them.*By this way, the average cold start overhead will be reduced, but caused resources waste. And, this method may cause some kind of reeducation of origin isolation.

However, there still two problems in this system:

1. Function Concurrency.

   No matter do concurrency in different containers or just wait a warm container in queue are all high-cost.

2. Function Chaining.

   For the internal and external chaining functions, existing platforms do not distinguish, causing undesired latencies.

### SAND

#### Application Sandboxing

The key idea is two levels of fault isolation :

1. isolation between different applications  
2. isolation between functions of the same application. 

Since contrasted with different applications, functions of the same application may not need such a strong isolation, allowing us to improve the performance of the application.

Different environments can be chosen: VM, LightVMs, containers, unikernels, processes, light-weight contexts and threads.

we specifically separate applications from each other via containers, such that each application runs in its own container. The functions that compose an application run in the same container but as separate processes, which brings three advantages:

1. forking a process within a container incurs short startup latency  
2. the libraries shared by multiple functions of an application need to be loaded into the container only once  
3. Resources can be released right after finishing functions, bringing the high efficient use of resources 

#### Hierarchical Message Queuing

Existing platforms often use a unified bus, which is not flexible enough, and even functions running in the same host need to interact through the bus.

Sand introduced the local bus on the basis of the global bus, so that the transmission of local information does not have to go through the global bus, which increases the efficiency. In actual operation, the global bus can be used as a backup of the local bus to improve system robustness,and take certain flags to guarantee exactly-once.

### System design

#### System Components

- In SAND, a function of an application is called a *grain*.
- sandbox : On a given host, the application has its own container called sandbox. The set of grains hosted in each sandbox can vary across hosts as determined by the application developer, but usually includes all grains of the application
- grain worker:  a dedicated OS process on sandbox

Through this operation, SAND achieves more fine-grained resource control, not only has less cost at startup, but also can be quickly released at low cost when resources are released 

#### Additional components

- Frontend Server.  
- Local and Global Data Layers  

### Evalution





 



