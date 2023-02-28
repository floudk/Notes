## Phoebe: A Learning-based Checkpoint Optimizer  

### Introduction

随着数据系统越来越大，在微软的实际环境中出现了一些新的问题：

1. 大型任务的临时存储会占用极大的空间，可能占用了所有SSD的内存
2. 大型作业错误后的恢复更需要时间
3. 大型作业的查询优化做的很糟糕

对于这些问题**checkpoint**都能有一个好的解决方案，一方面checkpoint可以减少临时存储，另一方面通过对checkpoint的分析可以进一步优化作业，最后checkpoint也可以进行恢复。

总的来说，具体贡献有四点：

1. 提出了Phoebe，一种基于学习的checkpoint决定器
2. 提出了accurate stage-wise cost predictors  
3. 提出了一个scalable checkpoint optimization algorithm
4. 对Phoebe进行了测评

### Background



### PHOEBE OVERVIEW

