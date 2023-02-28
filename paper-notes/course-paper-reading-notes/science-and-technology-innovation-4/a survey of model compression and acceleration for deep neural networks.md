### Abstraction

深度神经网络获得了巨大的成功，但也面临着参数量的急剧增长，这极大地限制了在资源受限情况下的应用。文章介绍当前主流的四种模型压缩方法，以及一些目前正在讨论的新方法，最后总结和展望该领域的未来。

### Introduction

一般来说，模型简化和加速的目的主要有两个：

1. 为了更快的计算，使得实时任务得到更好的支持
2. 为了资源受限情况下的应用。

主要的简化方法有四种：

- model pruning and quantization: 简化模型
- low-rank factorization：矩阵分解
- knowledge distillation：通过compact模型向原有模型学习
- transferred/compact convolutional filters：针对卷积网络，设计特殊的卷积层

这四种方法并部署孤立的，而是可以互相结合使用的。

### Parameter Pruning and Quantization

剪枝和量化在简化模型的同时还能增加模型的泛化能力，总的来说，又可以分为三个子类：

#### Quantization and Binarization

本质上就是降低表示精度，当只用1个bit来表示是，就是binarization.

#### Network pruning

本质上是去除网络中不那么具有表现力的权重，但也存在问题，剪枝会减速收敛，还需要人类更多的工作来进行调整，最后剪枝只能减小模型的规模，对推理速度的提升并不明显。

#### Designing Structural Matrix

从参数角度进行剪枝，直接将原有的大矩阵通过增加额外约束，变为structural matrix，从而获得更小的存储空间和更快的计算。但存在的我呢提也很多，一方面会损失精度，另一方面如何获得一个合适的结构化矩阵是困难且没有理论支持的。

### Low-rank approximation and Sparsity

>  tensor(张量)可以被理解为矩阵的泛化，实际上，矩阵是2阶张量。’
>
> **张量分解**从本质上来说是矩阵分解的高阶泛化。

总的来说，使用low-rank矩阵进行分解来压缩模型对于卷积层来说是非常经典的方法，在全连接层上也有一些研究进行支持。

但这种方法也面临着尴尬，逐层进行low-rank分解不光是计算昂贵的，同时还忽略了参数之间的关联

### Transferred/compact convolutional filters

使用特殊设计的filters来进行卷积，不光节省内存还可以加速计算，但一方面这种假设过于强，可能损失一些信息，另一方面，他对于深度网络的压缩效果也并不明显。

### Knowledge Distillation

其中之一是 KD 只能应用于具有 softmax 损失函数的任务，这阻碍了它的使用。 另一个缺点是，与其他类型的方法相比，基于 KD 的方法通常获得较少的竞争性能。

### 其他方法

1. attention-like mechanism
2. replacing the fully connected layer with global average pooling  
3. AUTOML

### benchmark



### 当前挑战

1. 压缩方法都适用于精心设计的CNN，自由度上有损失
2. 压缩方法没考虑过限制最大的硬件
3. 人类先验知识带来的影响
4. 压缩不具有可解释性

### 未来方向