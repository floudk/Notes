### Introduction

文章主要关心了在fog computing中与遇到的workload-resource匹配问题，重点解决了两个问题：

1. 什么时候进行扩展
2. 选择怎样的节点进行扩展

### 背景

1. DSP 应用在IoT时代是应用广泛的，但也面临着资源适配的问题，因此自适应的DSP系统得到了相当程度的发展。
   本文提出的系统主要基于MAPE (monitor, analyze, plan, execute)
2. Flink **back pressure**. Flink因为整体采用流线作业，一旦某个算子速度过慢，前面的算子会缓存自己的处理结果，同时减慢自己的处理速度，类似的效果被称为*back pressure*.
   因此，单纯的pressure 现象的出现无法判定某个算子是否overload，可能是由下游算子的overload带来的副作用，而应该是某条流向中，从overload开始往后没有back pressure现象。
3. Flink并没有设计为原生支持auto-scale的，尽管社区目前有这方面的考量，但相关工作还是并未展开。

### 相关工作

相当多的工作都在支持DSP弹性系统，但具体到不同的环境，部署方式，优化目标等还是相当不同的。

### 系统设计

#### 设计原则-MAPE

对于系统scaling优化分析来说，一种合理的分析方式是对单独的operator进行分析，每个operator单独的考虑。当然这种分析方式是否合理，需要针对具体的优化目标。本文中采取latency作为优化目标，所以考虑单独的operator的latency是合理的。

#### Monitor

以一分钟为周期监测backpressure等metric

#### Analyze  

分析主要决定一件事：什么时候进行rescaling？与此同时，需要完成一些参数矫正工作。

##### Performance model and its calibration

Performance model  的建模主要为了说明一件事，在给定operator和并行度的时候，该算子能达到的MST(maximum sustainable throughput)是多少

##### Deciding when to scale up

当观察到某个算子出现high back pressure的时候，既可以进行scale up

##### Deciding when to scale down

没有直接的指标说明资源过剩，考虑低CPU利用率也并不一定合理，因此我们考虑在降低并行度的同时计算MST，如果降低并行度后的MST仍然高于当前operator的throughput，那我们就可以合理的认为，是时候应该进行scale down了

#### Plan

在知道了需要进行rescaling之后，还有两个重要的问题

- 应该重新调整为多少
- 应该怎样调整
  在 fog computing场景下，网络的异构对性能影响是极大的，因此我们不光需要知道在什么时候如何调整并行度，还需要思考哪些节点更适合被添加或者删除。

##### scaling up

当出现back pressure的时候，我们知道至少需要增添1个新的节点，因此我们反复的增加一个新节点，观察是否需要继续添加即可。
此时，model的作用仅为当存在多个闲置资源的时候，我们应该选取哪个资源作为最佳的添加方式。

##### scaling down

scaling down的时候，可以基于performance model，在保持当前吞吐的基础上，尽可能多的减少节点。

#### Execute

执行阶段要做两件事情：

1. 获取资源列表

   需要从K8S获取可用的资源列表，同时应该按照网络延迟进行排序，当进行scale up的时候，应该尽可能选取网络延迟小的节点进行添加；当进行scale down的时候，应该尽可能选取网络延迟大的节点进行删除。

2. 执行reconfiguration

   还是老办法，先做savepoint，再进行rescale。