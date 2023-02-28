[GitHub](https://github.com/jbdel/MOSEI_UMONS)

### Background

Emotion recognition task has existed working on different types of signals typically *audio*, *video* and *text*.

*Deep Learning techniques* allow the development of novel paradigms to use these different signals in one model to leverage joint information extraction from different sources.

The main inspiration of this work is based on *Transformer* and *Modular co-attention*.

This work can compare with, and sometimes surpass the work proposed in *Multimodal Language Analysis in the Wild* on *CMU-MOSEI* dataset.

### Related work

Here, the paper lists several models that have been evaluated on the CMU-MOSEI dataset, and none of them uses a Transformer-based solution.

1. [*Memory Fusion Network*](https://openaccess.thecvf.com/content_CVPR_2019/html/Yu_Deep_Modular_Co-Attention_Networks_for_Visual_Question_Answering_CVPR_2019_paper.html) uses a *multi-view gated memory* that stores intraview and cross-view interactions through time.

   > Multi-view Gated Memory: Multi-view Gated Memory is a neural component that stores a history of cross-view interactions over time. It acts as a unifying memory for the memories in System of LSTMs. 

2. [*Graph-MFN*](https://aclanthology.org/P18-1208/) offers a *DFG* based network built upon *MFN*.
3. [*Tensor Fusion Network*](https://aclanthology.org/W18-3303/) uses an outer product of the modalities and are performer on frame by frame to save computation resources.
4. [*Multilogue-Net*](https://arxiv.org/abs/2002.08267) offers a solution based on a context-aware RNN.

### Model

Basically, the model is based on *Transformer* model, a new encoding architecture that fully eschews recurrence for sequence encoding and instead relies entirely on an attention mechanism.

The transformer allows for significantly more parallelization compared to RNN.

#### Monomodal variant

`Aim`: The monomodal variant is used to classify emotions and sentiments based **solely on** L (Linguistic), on V (Visual) or on A (Acoustic).   

The monomodal encoder consists of a stack of identical blocks. Each block has two sub-layers. And *residual connection* and *layer normalization* is used in this model.

$LayerNorm(x+Sublayer(x))$
In traditional Transformers, the two sub-layers are respectively a multi-head self-attention mechanism and a simple MLP.

<img src="imgs\monomodal transformer.png" alt="image-20220414155045330" style="zoom: 50%;" />



#### Multimodal variant

`Aim`:The multimodal version is used for **any** combination of modalities.  

Based on monomodal transformer encoding, the idea of a multimodal transformer seems natural to add a dedicated transformer for each modality. Besides this, the paper introduces three additional ideas:

1.  A joint-encoding
2. A modular co-attention
3. A glimpse layer at the end of each block, where the modality is *projected* in a new space of representation.  

$y=LayerNorm(y+MHA(y,x,x))$

#### Classification layer

A modality goes into a final glimpse layer of size 1, The result is therefore only one vector. Then sum up and project over possible answers according to the following equation.

$y~p=W_a(LayerNorm(s))$

### Feature extractions

This section is aimed to explain how we pre-compute the features for each modality, which are inputs of the *Transformer blocks*.

Note that the extraction is **independently** for each example of the dataset.

#### Linguistic

Lowercase and tokenize each utterance and remove special characters and punctuation.

Then use a vector of 300 dimensions to embed each word with *GloVe* and replace the unknown word with "unk".

#### Acoustic

In acoustic information, besides linguistic information, there are also *nonverbal* expressions(laughs, breaths,sighs) and *prosody* features(intonation,speaking rate).

Although some _handcrafted_ sets of high-level features has been generally recognized in speech processing field, these methods indeed discard a lot of information.

So a low-level features called **mel-spectrograms** are used in speech processing. Mel-spectrogram is
a compressed version of spectrograms, using the fact the human ear is more sensitive to low frequencies than high frequencies.  

#### Visual

Use a pre-trained CNN, consisting of a 2D spatial convolution and a 1D temporal convolution.

Notice that, in this paper, we do not *crop out the face region* of the video and keep entire image as input, believing the movement of body and other indicators can be useful for emotion recognition and sentiment analysis tasks.

### Dataset

Each sentence is annotated for sentiment on a [-3,3] scale from highly negative (-3) to highly
positive (+3) and for emotion by 6 classes : happiness, sadness, anger, fear, disgust, surprise.

In the experiment, it is a binary classification whether or not existing emotions, but two emotions can existed at the same time.

### Results

The results show that L+A is the best model, and audio has roughly no effect on multi-modal tasks, which shows that it is still a challenge to do model fusion.

-----

### 问题

1. Emotion Recognition 和 Sentiment Analysis 的区别是什么？

   Sentiment Analysis旨在从文本中检测出积极、中性或消极的感受

   Emotion Recognition旨在通过文本的表达来检测和识别感受的类型

2. Attention机制是什么意思？ [【Attention九层塔】注意力机制的九重理解](https://zhuanlan.zhihu.com/p/362366192)

   Attention实际是一种帮助模型有效地“注意到”问题相关的特征中。

   - 基于存储器的实现：最早的注意力机制可以通过存储器实现，这种思路是简单直接地，通过对存储器的处理与更新来实现注意力的转移，但方法不是那么的Fancy.

   - Attention是一种加权平均

     $Attention(Q,K,V)=softmax(\frac {QK^T}{\sqrt d_k})V$

     公式的本质是*按照关系矩阵进行加权平均*。

   - Transformer: Attention is all you need

     相比recurrent，Attention不许进行序列操作，使用空间换取时间，尽管每层复杂性变大了，但计算量减少；相比convolution也能有效地减少计算量。

   - BERT: Attention新高度

     BERT创造性地提出在大规模数据集上无监督预训练加目标数据集微调（fine-tune）的方式，采用统一的模型解决大量的不同问题。

   - CV: Attention是有效的非局域信息融合技术

     attention本身是一个加权，加权也就意味着可以融合不同的信息

     通过讲CNN和Attention结合的方法，改善了原本CNN中卷积只能提取出局部信息的问题

   - CV: Attention will be all you need.

     能否设计纯transformer的网络做视觉的任务? 

     *Vision Transformer*(Google) 通过将图片编码成文字，再输入Transformer，将其应用于图像分类。

   - 结构化数据：Attention是辅助GNN的利器

     利用Attention可以对高维数据进行处理

   - 逻辑可解释性与Attention的关系

     Attention这种加权的分析，天然就具有可视化的属性。而可视化是我们理解高维空间的利器。

   - Attention的多种变种以及内在关联

     attention的最厉害的地方，是它可以作为基本模块搭建起非常复杂的模型

3. Transformer简单的说是在做什么?

   Transformer 本质上是一个*编码-解码器*, 抛弃了传统的CNN和RNN，整个网络由*Attention*机制组成。

   - *Encoder*:把输入映射为隐藏层， 即含有自然语言序列的数学表达。

     Transformer没有RNN的循环结构，因此需要使用额外的方式来捕捉顺序序列

     self-attention

     残差设计和层归一化

     前馈网络

   - *Decoder*: 把隐藏层还原为和输入形式一致的的输出

4. MHA多头注意力是什么？*自注意力*是什么？

   *多头注意力*就是对同样的Q, K, V求多次注意力，得到多个不同的output，再把这些不同的output连接起来得到最终的output。

   当注意力的query和key、value全部来自于同一堆东西时，就称为*自注意力*。