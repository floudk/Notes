### Introduction

For serverless, High level elasticity involves extra latency associated with on-demand provisioning of individual runtime containers, that's to say :**cold start**. On contrary, if a container is ready to serve requests with close to zero overload, we call it *warm start*.

To solve cold start problem,  A common approach is to maintain several *warm containers* to serve future requests.

This paper implement a pool of function instances and evaluate the latency compared with original implementation, showing lower latency.

#### Serverless

serverless allows the developer to focus on the functionality of the service itself without worrying about any environment and system issues, such as scaling and fault tolerance, providing a simple-to-deploy model for developers.  

#### Cold start

Cold start occurs not only in spinning up the first instance of functions, but also in auto scaling. The initialization overhead is unavoidable and known as **cold start**, although this problems naturally exist in FaaS, however the concrete delay is based on the implementation in different structures.

For *Kubernetes*, the main overhead can be split into two categories:

1. Platform overhead, including network bootstrapping or so, this depend on the platform itself. And for the same platform, different functions get uniform costs in platform overhead.
2. Application-dependent initialization overhead: Any overhead caused by the application, including loading needed libraries.

The Paper comes a way to reduce cold start overhead with maintaining a warm pod pools, so we do not need cold start anymore for future invocation. Additionally, Since the focus is just functions, so the pools can be shared among different services using same functions.

### Implementation

#### Pods Pool

The *Knative* natively support pods pool underlying structure.

And in native Knative, Sidecar containers in the pod  will send control messages to the autoscaler  and trigger the reconciliation in Kubernetes to create/destroy pods in the replicaSet.

The paper do a extra check : Before changing the desired number of pods in the replicaSet, we first check if there are any pods in the pool that can be migrated to the target, to reduce the costs, which takes almost 2 seconds for migrating.

#### Evaluation

1. Response Time to First Request 

   use two different applications for evaluation

   - HTTP : save approximately 7 seconds
   - Tensorflow image classifier :  save approximately 33 seconds

2. Simulation on Traces  

   Test the system performance in a more realistic, large-scale scenario  

   *Add a pool significantly improves the overall response time* , even there is only 1 pod in pool.

#### Response Time for Fixed Pool Size  

It seems as number of pods in the pool increases, the total benefits will decrease, and 5 pods may be enough for solving cold start problem. 

### Related work

Serverless is really a new area with less than 8 years. And most details of serverless originated from the industry, which means they differs a lot in different implementations and not open source.

1. OpenWhisk  use a prewarm mechanism, [Apache](https://www.google.com.hk/url?sa=t&rct=j&q=&esrc=s&source=web&cd=&cad=rja&uact=8&ved=2ahUKEwikjNzslqL1AhVYPnAKHapSAsoQFnoECAUQAQ&url=https%3A%2F%2Fmedium.com%2Fopenwhisk%2Fsqueezing-the-milliseconds-how-to-make-serverless-platforms-blazing-fast-aea0e9951bd0&usg=AOvVaw2TAefZD6ozJ41oZTkPyulq)

   Use system cache for database and also for containers.

   To reuse a container, we need to do container prewarm, not hot. But warm at least.

   Use some a priori guesses about user needs, preload some libraries and environments to achieve warm-up, fitting well for stateless situations.

2. Serverless Computing: Design, Implementation, and Performance  

   Use global cold queues that have pre-allocated empty containers ready to be initialized as any function, and a warm stack per function that reuses previous containers. This approach provides the flexibility to reduce platform-wise cold start overhead, but the application overhead cannot be mitigated.

3. SAND: Towards High-Performance Serverless Computing
   Use a higher level abstraction runtime instead of containers to reduce the costs. How to make good isolation is still a problem.
   
4. Some computer science problems similar to cold starts 

   1. The "prefetching/prewarming" concept also originates from **cache** designs.

      For cache problem, temporal locality and spatial locality can be useful. However, in cold start problem, only temporal locality are considered, many approaches leveraging spatial locality are invalid in our context. Still , contrasted with cache, instances are mutually exclusive and cannot be shared.

   2. **knapsack problem** 

   3. **Inventory models** 

      Cold start is very similar to the inventory model in the field of management.

### Future work

1. Sharing pools among multiple services  
2. More realistic simulation
3.  Introduce mathematical model, such as *inventory model*.