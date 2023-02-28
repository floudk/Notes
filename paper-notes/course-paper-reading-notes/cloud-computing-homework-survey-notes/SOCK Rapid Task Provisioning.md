### Introduction

SOCK, a special-purpose container system with two goals:

1. low-latency invocation for Python handlers that import libraries   
2. efficient sandbox initialization  

And based on 3 techniques:

1. lightweight isolation primitives  
2. Python handlers using a generalized Zygote-provisioning strategy  
3. leverage our generalized Zygote mechanism to build a three-tiered package-aware caching system  

### costs of Linux provisioning primitives  

Serverless platforms often isolate lambdas with containers.

Prepare a file system for a container need two steps:

1. populate a subdirectory of the host’s file system with data and code needed by the container  
   
   Populating a directory by physical copying is prohibitively slow, so all practical techniques rely on logical copying
   
2. make the subdirectory the root of the new container  for the isolation.

Three conclusions from the observation:

1. in a serverless environment,all handlers run on one of a few base images, so the flexible stacking of union file systems may not be worth to the performance cost relative to bind mounts
2. network namespaces are a major scalability bottleneck  
3. , reusing cgroups is twice as fast as creating new cgroups  

### Python Initialization Study

language runtimes and package dependencies can make cold start slow.

Strong popularity skew further creates opportunities to pre-import a subset of packages into interpreter memory

### SOCK

1. we want low-latency invocation for Python handlers that import libraries  
2. we want efficient sandbox initialization so that individual workers can achieve high steady-state throughput. hides latency by maintaining pools of pre-initialized containers  

We do the following things:

1. build a lean container system for sandboxing lambdas  

   - **Storage**  SOCK uses bind mounts to stitch together a  root from four host directories. We use the faster and simpler chroot operation  
   - **Communication** :
   - **isolation**: Mount namespaces are unnecessary because SOCK uses chroot  

2. generalize Zygote provisioning to scale to large sets of untrusted packages  

   Zygote provisioning is a technique where new processes are started as forks of an initial process:

   1. the set of pre-imported packages is determined at runtime   
   2. SOCK scales to very large package sets by maintaining multiple Zygotes with different pre-imported packages,
   3. provisioning is fully integrated with containers
   4.  processes are not vulnerable to malicious packages they did not import

3. three-layer caching system for reducing package install and import costs  

   1. , a handler cache maintains idle handler containers in a paused state and Paused containers cannot consume CPU, and unpausing is faster than creating a new container; however, paused containers consume memory,so SOCK limits total consumption by evicting paused containers from the handler cache on an LRU basis.
   2. an install cache contains a large, static set of pre-installed packages on disk.
   3. SOCK decides the set of Zygotes available based on the import-cache policy.