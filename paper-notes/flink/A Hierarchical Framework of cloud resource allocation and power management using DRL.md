### Introduction

Automatic decision-making approaches, 比如reinforcement learning已经在云计算领域运用的十分广泛了，但过去系统对状态空间有一定限制，所以往往无法利用好DL模型进行资源分配，同时考虑到电源问题，我们提出一个**hierarchical**的框架来解决云计算的资源分配和电源管理问题。

由一个*Global tier*来分配计算资源，并且有一个*Local tier*来做分布式电源管理,总的来说

- Global tier是通过deep reinforcement learning支持的，可以对具有极大状态空间的复杂问题进行求解，并且额外的autoencoder和weight sharing stucture用来处理高维的状态空间和加速收敛
- Local tier是由基于LSTM的workload predictor和一个基于model-free RL的电源管理机制

### Background

RL模型是适合做这类问题的，因为RL模型不需要任何前置的模型知识，是model-free的，同时可以通过系统运行中的在线学习。

传统的RL受制于云资源分配系统中状态空间过大，往往是简单地对电源进行管理，或者对相似的资源进行管理。而emerging DRL  适合于做这样的工作。

同时DPM-dynamic power management 对未被运用的机器进行管理也被证明是云计算中减少电源消耗的常用技术，DPM技术也使用了很多机器学习技术。

一个自然的想法就是将资源分配和DPM融入一个系统，但与此同时，使得状态空间更大了，因此需要考虑一个新的解决方案。

技术上来讲，DNN可以被用于从高维的输入中抽取信息，Q-learning 可以进行在线的状态中进行学习。

- RL 环境

  一个经典的RL或者DRL模型由以下部分组成
  | 名称                 | 作用                                                         |
  | -------------------- | ------------------------------------------------------------ |
  | agent                | 与环境交互，每个epoch基于当前状态$S_k$进行动作$a_k$, 动作的目的是追求反馈的最值 |
  | environment          | 对agent的每个动作产生反映，生成新的环境状态$S_{k+1}$,同时需要提供反馈$r_k$ |
  | finite state space S |                                                              |
  | available actions    |                                                              |
  | reward function      |                                                              |

- Continuous-Time Q-Learning for Semi-Markov Decision Process(SMDP)

  针对半马尔可夫决策的Q-Learning算法。
  

### 系统模型和问题定义

系统分为global tier和local tier

- 全局层主要是存在一个*jobs broker*会对任务进行分配，将任务分配给对应的server，每个server采用FCFS的方式进行服务，如果当前的资源无法对指定任务进行服务，那么该server会在当前任务完成后资源充足时继续执行任务

- local tier主要是一个决定是否将当前server由active模式转入sleep模式的选择器。当server在sleep mode下时，耗电量为0，在active模式下，耗电量与当前CPU的占用率正相关。同时模式转移也存在耗电，因此频繁的关闭和开启server不光会导致耗电量的增加，还会导致延迟的提升。

  一个常用的办法就是*DPM*技术。DPM可以以一种online-adaptive的方式决定最佳的timeout。并且DPM也需要global tier的配合，比如一个*workload predictor*就是必不可少的。也就是说*DPM*技术的优劣和预测器的好坏是紧密关联的。一些经典方法采用了贝叶斯方法，但如今，考虑RNN，LSTM等方法都是更合理的的选择。

总的来说，global tier使用了*emerging DRL*技术用来减少状态的数目，而local tier使用*RL*方法配合一个*LSTM*网络。

### Overview of Deep Reinforcement Learning

总的来说，DRL采用的方法是，先使用一个offline的方法预训练出一个DNN网络，同时在online的过程中使用该DNN网络边预测边进行改进。

相对来说，它能支持很大甚至是连续的状态空间，但因为它每次试探都要对所有的可能做试探，因此需要限制状态空间的维度在一定的限度内。

### DRL-Based Cloud Resource Allocation



