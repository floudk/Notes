### introduction

A unique feature of Serverless computing is the ability to scale to zero. It means, after triggering a function by an event, function will be deployed on a container and given all the resources needed to execute it.  When the function executed completely and there were no subsequent requests, all the
allocated resources will be released.  

The serverless elasticity born with **cold start** problems, since the runtime container should be  supplied. To respond to future requests, resources must be reallocated to the function. This process
takes a period of time known as cold start delay.  

To reduce the cold start costs, two main approaches are introduced:

1. Reduce the cold start delay.

   There are two general solutions to reduce this delay  :

   1. *Prepare a container for each function's running*: will increase idle functions costs including CPU and memory overhead.
   2. *Pause the temporary unused container*: reusing paused container has lower delay than assigning a new cold container.
   3. *Use a container Pool*: The method is not cost and resource effective [fission](https://fission.io/docs/architecture/executor/) 
   4. *Keep containers warm after executing* :keep the container warm to respond to any subsequent requests such as AWS and Google. However, this may cause wasted resources.
   5. *Using two hot and cold container queues*, However I am doubting this method may be same as container pool in essence. 
   6. *Application level sandboxing*. Just like the container is high level of OS, we can use a higher abstraction in user level. In this way, all functions in same program can run in same container with different processes, which has lower cost than container. However, in this case, **Isolation** can be a very hard problem in the same container.
   7. *Preloading required libraries*. This method is a little same as the above one, preload the needed libraries is just like we abstract in a higher level.

2. **Reducing the delay of loading functions' libraries.**
	1. *preventing containers from getting cold*. Used to schedule functions to ping periodically and trigger functions in a timely manner and prevent them from getting cold and idle. 

### Evaluating methods

According to experiments, the amount of **memory** and **programming language** affect the cold start delay. However, with different implementations, the main influencing factors may differ.

In the experiments, I/O-Intensive functions have a longer cold start delay than CPU-Intensive functions since bigger libraries and extra file needed to be installed into the container.