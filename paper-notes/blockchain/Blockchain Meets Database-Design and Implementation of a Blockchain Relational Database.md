# 阅读笔记

## 整体印象(10-15min)

1. 基本信息  
阅读标题、摘要、引言

2. 阅读每个标题和子标题

3. 扫描理论推到部分

4. 阅读文章结论部分

5. 扫描参考文献，重点标注较新的重点顶级刊物文章

|标题|作者|3个以内的关键词|是否解决核心问题|
|----|----|-------------|---------------|
| Blockchain Meets Database: Design and Implementation of a Blockchain Relational Database | Senthil Nathan(IBM) | Relational Database;<br />Blockchain;<br />consensus | 通过改良现有协议实现了基于blockchain的拜占庭条件下的多关系型数据库 |



|文献名称|会议或期刊名称|是否考虑阅读|
|----|-----|-----|
| Hyperledger fabric: a distributed operating system for permissioned blockchains | EuroSys 18 | **是** |
| Veritas: Shared verifiable databases and tables in the cloud | CIDR 2019 | **是** |
| Xft: Practical fault tolerance beyond crashes. | USENIX  2016 | **暂不** |
| Performance benchmarking and optimizing hyperledger fabric blockchain platform. | MASCOTS 2018 | **否** |
| When query authentication meets fine-grained access control: A zero-knowledge approach. | SIGMOD  2018 | **暂不** |

**回答下列问题**  

- 文章的类别是什么?
> Conference Paper，VLDB2019会议论文呢

- 文章背景是什么?
> 背景基于两点：
>
> 1. blockchain如火如荼，尽管有人认为blockchain是跨时代的技术，但还有从数据库角度审视该技术的观点。而且目前blockchain很多方向冀望更新的技术来获得一些特性，但实际上这些新特性在关系型数据库中已经得到了。
> 2. 在一组不受信任的组织中，每一个组织享有一个数据库副本，如何在拜占庭条件下实现共识，这就是本文主要的假设。

- 文章准确性如何
> 暂时还没有定量的描述，但定性的提出了问题和解决方案。

- 文章创新性如何
> 文章用了first-ever来描述，但也有对现有协议的更改，总体来说解决的问题还是挺好的

- 文章描写是否是易于理解的
> IBM 的文章，写的还是相当清晰明了的

**决定如下情况**
- 是否要继续阅读

  要继续阅读

- 是否应该精度

  要精度，blockchain领域对我还是陌生的，边读便补充背景

- 是否有感兴趣的地方

  - blockchain 可以看做数据库，有对这一观点更详细的说明么？
  - blockchain本质上的实现是什么，拜占庭的引入对非拜占庭的一致性算法有哪些影响呢？

## 深刻把握内容

### 文章结构

#### Motivation

阐述数据库和区块链的异同

#### Design

先给出概念，主要包括一些背景组件和协议，然后最事先明确sequence和不明确sequence的两种情况提出方法

最后对方法更进一步讨论security、recover和network问题

#### Implementation

基于PostgreSQL实现，先简单介绍sql和其他组件，然后介绍一些定制化操作主要是对组件的修改

##### Evaluation

设计实现在多个层面对实现进行评估

#### Related Work

给出并简要介绍相关工作

#### Conclusion

对全文的总结和讨论






## 深入感兴趣的内容点
