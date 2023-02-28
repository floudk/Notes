## A simple and effective predictive resource scaling heuristic for large-scale cloud applications  

### Introduction

资源的auto-scaling是必要的，既能够避免资源不足带来的程序无法正常运行，也能够一定程度上节省资源。

本研究有三点贡献：

1. 给出了一种方法来分析和评估auto-scaling策略
2. 展示了*probabilistic forecasts* 能够满足对风险的规避
3. 通过实验说明了简单的*启发式*方法在极高的风险规避情况下能够达到最优状态。

### related work

> **Reactive Scaling**: 系统基于监测的指标，维持资源在某个上限和下限之间，往往是有扩容和缩容的触发条件。
>
> **Proactive Scaling**：基于计划的rescaling，一般是workload模式比较确定的作业
>
> **Predictive Scaling**：基于预测的扩展，一般是通过预测器

### 问题定义

基础假设：

- 使用每台服务器接收和发送的bytes数总和作为metric
- 认为所需要的hosts数目$z_t$和$\sum v_t$成线性关系
  （在实际中，可能需要更加复杂的模型）

#### Probabilistic Forecast  and cost function

损失函数主要来自于两个方面

1. 硬件损耗，这里看作与host数目成线性关系
2. service's performance，即客户对服务的满意度。这样的往往是难以度量的,为了简化问题，可以做出如下的假设：假设存在一个最优的资源分配$r^*$,如果分配资源超过$r^*$,那么可以认为其不会随着资源更多而增长，而当不够时，则不够的越多，cost越大。

但$r^*$的估计是困难的，因此采用$Z_t$进行估计。

需要额外注意的是$Z_t$是一个随机变量，在运算中要实际使用期望来衡量cost function的最小值。

#### Resource Provider

对于资源提供商来说，基本假设有以下三个：

1. 资源是无限的
2. release资源是瞬时的
3. 资源request是有限制的，我们将限制简化为如下的模型：资源供应商可以有一定量的启动器，用于启动新的实例并下载对应packages，一旦启动器全部在工作中，那么新实例需要等待启动器

#### Scaling Cost Model

