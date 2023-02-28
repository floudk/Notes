### Introduction

A benchmark was developed for Container to conduct comprehensive testing. And based on the test, a new storage driver is designed to accelerate the startup of the container. 

Containers are essentially just processes that enjoy virtualization of all resources, not just CPU and memory; as such, there is no intrinsic reason container startup should be slower than normal process startup. Which is much slower in practice since the require of a fully initialized file system, containing application binaries, a complete Linux distribution, and package dependencies. 

To resolve this problem, the HelloBench is post and through this we found:

1. copying package data accounts for 76% of container startup time  
2. only 6.4% of the copied data is actually needed for containers to begin useful work  
3. simple block-deduplication across images achieves better compression rates than gzip compression of individual images.

With these findings, we build the Slacker, a *Docker storage*, utilizing specialized storage-system support at multiple layers of the stack  to reduce the overhead, which uses the snapshot and clone capabilities to do this.

At the same time, we use a lazy way for Slacker to request the image, and use a modified Linux kernel to improve the cache sharing.

### Docker

The default storage driver is AUFS, which se another file system (e.g., ext4) as underlying storage instead of directly on disk, which at file granularity has some performance problems.

### Hellobench

This benchmark is designed in a very exhaustive way to test about the following questions:

1. how large are the container images, and how much of that data is necessary for execution   

   For a container, the image compression pass is expensive and not fully used, which is not worth it. Therefore, it can be considered to acquire only when needed, and compress and acquire the image as a module, which is far more efficient than a complete image. 

2. How long does it take to push, pull, and run the images  

   Usually, the deployment of a docker would be in a central registry way. And Thus, 76% of startup time will be spent on` pull` when starting a new image hosted on a remote registry.

   Finding ways to optimize major time-consuming operations can greatly improve the performance of the entire process 

3. How is image data distributed across layers, and what are the performance implications, showing `run` are fast while pushes and pulls are slow.

   Image data is typically split across a number of layers and driver  driver composes the layers of an image at runtime to provide a container a complete view of the filesystem.

   for layered file systems, data stored in deeper layers is slower to access. Unfortunately, Docker images tend to be deep, with at least half of file data at depth nine or greater. Flattening layers is one technique to avoid these performance problems, which however has other problems.

4. how similar are access patterns across different runs  

   In the multiple use of the container, the cache will have a considerable amount of optimization for the repeated data, but for large systems, because the possibility of repeated data is very small, it is basically useless. 

### Slacker

   The following goals need to be satisfied: 

1. make pushes and pulls very fast  
2. introduce no slowdown for long-running containers  
3. reuse existing storage systems whenever possible  
4. utilize the powerful primitives  provided by a modern storage server 
5. Make as few changes as possible to ensure compatibility 

#### Storage Layers

1. Container uses only 6.4% image data to run, and request lazily if needed. And at the same time use a NFS  file system to support this.
2. Our analysis showed that multiple containers started from the same image tend to read the same data, suggesting cache sharing could be useful