### Introduction

Use .NET to implement a serverless system with Windows containers in Microsoft Azure, and evaluate the implementation with existed serverless systems.

The most interesting part in this paper may be the physical details of implementation.

### Serverless benefits

Serverless computing has proved a good fit for IoT applications, intersecting with the edge/fog computing infrastructure conversation.  

Serverless computing is often championed as a cost-saving tool.

### Implementations

The system are based on Azure platform, consisting of two components: *web service* and *a worker*.
A worker will manage and execute function containers and the web service will be used to find worker. Additionally, Function code and metadata are all stored in Azure Storage.

#### Function

1. metadata 

   - *Function Id* : GUID assigned during function creation and used to identify and locate functions
   - *Language Runtime*: The language and related support information of the function's code
   - *Memory size*: the maximum memory a function’s container can consume  
   - *Code Blob URI*: where system storage the code of function during runtime.

2. Execution

   This implementation simplifies the execution related operations since just manual execution support is just sufficient for our aims.

   When invoking a function. the web service will receive and retrieve related metadata and then create execution request consisting of metadata and inputs, and will find an available container to process the execution request.

   Specifically, *there is a global ”cold queue”, as well as a ”warm queue”* for each function in the platform, which includes the information about available container messages. *Cold queue* holds all information about not initialized containers now, which can be initialized later for using, and the *Warm queue* holds these that are currently able to use but do not hand any execution requests. All requests will first find any available containers in warm queue.
   
   - Container creation
   
     When creating a container, the allocation information will be firstly sent to the cold queue and not yet been assigned to a function. After a worker service receiving commands and allocate the memory, the container can provide the runtime for the function. And after function execution, the container will enter into *warm queue*.
   
   - Container Removal
   
     1. The function is deleted, and the functions' warm queue will be deleted.
     2. Idle for 15 minutes which is set arbitrarily.
   
     And the container will be sent to *cold queue* if the memory is enough for the function needed.
   
3. Container image

   Use Docker to run Windows Nano Server, which only includes read-only codes and excludes other functions in the container. By the way, windows container is much larger than linux one, so more cost is needed.

### Results

To test the architecture, the paper designs a series of measurements with some platforms, including AWS Lambda, Azure Functions, Google Cloud Functions and Apache OpenWhisk. Then the writer does the following tests:

1. Concurrency test.

   To measure the ability of serverless platforms to performantly invoke a function at scale.  

2. Backoff test

   To study the cold start times and expiration behaviors of function instances in the various platforms. The backoff test sends single execution requests to the test function at increasing intervals, ranging from one to thirty minutes.

### Limitations and Future work

#### Warm queue

The FIFO warm queue will lead to a lot of computational redundancy, which can be much better for using the *warm stack* in FILO.

#### Asynchronous Executions

The difficulty in asynchronous execution is in guaranteeing at-least-once execution rather than best effort
execution。

#### Worker Utilization

Since not all functions are constantly executing or using all computing resources, It's useful for workers to improve utilization. However, this is difficulty for needed for large-scale function load data.

#### Windows containers

Linux container is more powerful than the windows container for the support of container *resource updating* and *container pausing*.

#### Security and Performance

