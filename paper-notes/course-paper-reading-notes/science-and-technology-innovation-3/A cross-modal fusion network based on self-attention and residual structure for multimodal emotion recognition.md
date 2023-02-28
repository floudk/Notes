## Introduction

Typically, multimodal emotion recognition can be classified according to **the modal information fusion method**:

1. early fusion: Extract and construct multiple modal data features into one modal feature.
2. late fusion: Find out the method to coordinate and make joint decisions of each models.
3. model fusion: Model fusion is often done using **Transformer** for cross-modal interactions.

The defects of existing methods:

- Ignore the complementary information between different modalities. Decisions are always made just by two modality-specific features and fused features.
- Model stitching methods are redundant and do not reduce duplicate representations.
- The learning of intra- and inter-modal information will lose some semantic information.

So this paper proposes *CFN-SR*, a novel cross-modal fusion network based on self-attention and residual structure:

1. Perform representation learning for audio modality and video modality.
2. Feed features into cross-modal blocks separately, and use a self-attention mechanism to make the audio modality perform intra-modal feature selection. The residual structure are used in this process to guarantee the integrity of original features.
3. Obtain the output of emotions by splicing the obtained fused representation and the original representation.  

While having roughly identical performance with existing state-of-art models, CFN-SR can be more effective.



## Methodology

<img src="imgs\CFN-SR structure.jpg" alt="image-20220406150857280" style="zoom:50%;" />

### Audio Encoder  

For audio modality, the deep learning methods based on MFCC features have been widely recognized. In this paper, the authors roughly do the following steps:

1.  Use the feature preprocessed audio modal features as input, denoted as $X_A$  

2. Then pass the MFCC features through a 2-layer convolution operation to extract *the local features* of adjacent audio elements. 

   $\bar X_A = BN(ReLU(Conv1D(X_A,k_A)))$

   - $BN$ stands for Batch Normalization  
   - $k_A$ is the size of the convolution kernel of modality audio
   - $\bar X_A$ denotes the learned semantic features  

3. Then, we use the *max-pooling* to downsample compress the features and remove redundant information

   $\bar X_A=Dropout(BN(MaxPool(\bar X_A)))$

4. feed the learned features into a 1D temporal convolution to obtain the higher-order semantic features of the audio. And finally flatten it.

   $\bar X_A = Flatten(BN(ReLU(Conv1D(\bar X_A,k_A))))$  

### Video Encoder  

Video data are dependent in both spatial and temporal dimensions.

 The paper uses **3D ResNeXt** network, a network with a simple structure but powerful performance, to obtain the spatio-temporal structural features of video modalities.

1. Use feature preprocessed audio modal features as input, denoted as $X_V$

2. Obtain the higher-order semantic features of video modalities with ResNeXt:

   $\bar X_V = ResNeXt_{50}(X_V)\in R^{C\times S \times H \times W}$

   - $\bar X_V$ denotes the learned semantic features  

   - C, S, H and W are the number of channels, sequence length, height, and width, respectively.  

After getting video and audio features, system feeds them into the cross-modal blocks and fuse the audio feature representations. And then performs downsampling using the average pool to reduce redundant information.

### Cross-modal Blocks

After obtaining higher-order semantic features for both audio and video modalities, we need to exploit the complementary intra- and inter-modal interaction information between the two modalities.

1.  Make the audio modality undergo intramodal representation learning, and make it more focused on features that have a greater impact on the outcome weights.
2. Following Transformer, we use *self-attention* to perform feature selection on audio features.

### Classification

Finally, we can use the fused representation and original representation to do prediction. And we can use the cross-entropy loss to optimize the model.

## EXPERIMENTS  

### Datasets  

- RAVDESS  includes 8 emotions: neutral, calm, happy, sad, angry, fearful, disgust and surprised.

### Implementation Details  

- Video modality: Perform face detection on the basis of video sampling, crop it to a uniform size, and then perform data enhancement
- Audio modality:  Crop audio to the same length.

Based on the above measures, we perform 5-fold cross-validation on the RAVDESS dataset to provide more robust results.

### Baselines

1. Typical early fusion methods: MCBP and 'Multimodal fusion' to do simple feature concatenation.
   [Multimodal Compact Bilinear Pooling for Visual Question Answering and Visual Grounding](https://arxiv.org/abs/1606.01847)
   [Multimodal Fusion with Deep Neural Networks for Audio-Video Emotion Recognition](https://arxiv.org/abs/1907.03196)

2. *Averaging* and *multiplication*, as two standard late fusion.

3. *Multiplicative layer* with a down-weighting factor to CE loss.

   [Learn to Combine Modalities in Multimodal Deep Learning](https://arxiv.org/abs/1805.11730)

4. *MMTM* module allows slow fusion of information between modalities by adding to different feature layers,

   [Mmtm: Multimodal transfer module for cnn fusion](https://openaccess.thecvf.com/content_CVPR_2020/html/Joze_MMTM_Multimodal_Transfer_Module_for_CNN_Fusion_CVPR_2020_paper.html)

5. *MSAF*:

   [MSAF: Multimodal Split Attention Fusion](https://arxiv.org/abs/2012.07175)

6. *ERANNs*:

   [ERANNs: Efficient Residual Audio Neural Networks for Audio Pattern Recognition](https://arxiv.org/abs/2106.01621)

> Chinese Edition:
> [A CROSS-MODAL FUSION NETWORK BASED ON SELF-ATTENTION AND RESIDUAL STRUCTURE FOR MULTIMODAL EMOTION RECOGNITION ](https://github.com/skeletonNN/CFN-SR)
>
> ### 背景
>
> 当前跨模态的情绪识别方法可以按照模型信息交互的方式分为三种：
>
> 1. early fusion: 预先将不同模型的信息抽取并整合为一个统一的feature进行处理
>
>    ([Multi-Modal Emotion recognition on IEMOCAP Dataset using Deep Learning](./Multi-modal emotion recognition on IEMOCAP with neural networks.md)
>
> 2. late fusion: 每个模型独立的进行决策，然后将决策信息按照某种方式进行整合([Learning Affective Features With a Hybrid Deep Model for Audio–Visual Emotion Recognition](https://ieeexplore.ieee.org/abstract/document/7956190), 
>    [Multitask Learning and Multistage Fusion for Dimensional Audiovisual Emotion Recognition](https://ieeexplore.ieee.org/abstract/document/9052916) )
>
> 3. model fusion: 使用Transformer进行模型之间的交互
>
>    ([Multimodal Emotion Recognition with Capsule Graph Convolutional Based Representation Fusion](https://ieeexplore.ieee.org/abstract/document/9413608), 
>    [Multimodal Cross- and Self-Attention Network for Speech Emotion Recognition](https://ieeexplore.ieee.org/abstract/document/9414654))
>
> 现存方法遇到的主要问题：
>
> 1. 忽略了模型之间feature的互补，简单的使用了独立的模型feature和fused feature进行决策
> 2. 未在fused的时候去除掉不同modal中冗余的信息
> 3. 在模型内和模型间都会丢失信息语义
>
> 文章提出了基于*注意力机制*和*残差网络*的CFN-SR，采用modal fusion的方式，可以在与同类的解决方案表现一样好的同时，更加高效。
>
> CFN-SR大致使用如下步骤：
>
> 1. 将视频、音频信息进行编码，并输入网络。分别通过*3D CNN*和*1D CNN*网络来获取视频和音频的features
> 2. 将feature输入到各自的modal，并通过注意力机制进行features的选择性交互，在此过程中，利用残差网络保证原始feature 结构的完整性
> 3. 基于原始结构和网络得到的结果综合做出emotion的判定
>
> ### CFN-SR
>
> <img src="imgs\CFN-SR structure.jpg" alt="image-20220406150857280" style="zoom:50%;" />
>
> 总的来说，
>
> - **音频模型**上已经有了公认的有效处理方法：基于MFCC的深度学习。因此这里也选用MFCC作为特征选择方法：
> 1. 音频预处理: $X_A$
>   2. 将MFCC特征传递给2层的卷积层，以提取相邻音频之间的local features
>
> 3. 使用max-pooling做降采样，以压缩features，并去除冗余的信息
>
> 4. 将其输送给1D的时序卷积以获得更高维的semantic feature,最后将其flatten.
>
> - **视频信息**是与空间和时间都相关的，基于3D ResNeXt 网络，能够很好的得到视频在时空上的高维特征。
>
>   在得到视频、音频的特征后，将二者送入cross-modal blocks以进行fuse，最后进行降采样来去除冗余信息
>
> - **Cross-modal Blocks**
>
>   总的来说存在2步，
>
>   1. 使用*Transformer*做模型间的学习交互
>   2. 使用*self-attention*做特征选择
>
> - **Classification**
>
>   基于残差网络，使用上面结果和初始结构共同做情绪分类
>
> ### 数据集操作
>
> 使用**RAVDESS** 数据集，包含了8种情绪的音视频，
>
> 音频信息统一裁剪为相同长度
>
> 视频信息进行采样后，裁剪为相同的size，并基于基本的变换做数据增强
>
> 使用5-fold 交叉验证
>
> ### 相关工作
>
> 1. 经典early fusion跨模态模型 MCBP and 'Multimodal fusion' .
>    [Multimodal Compact Bilinear Pooling for Visual Question Answering and Visual Grounding](https://arxiv.org/abs/1606.01847)
>    [Multimodal Fusion with Deep Neural Networks for Audio-Video Emotion Recognition](https://arxiv.org/abs/1907.03196)
>
> 2. *Averaging* and *multiplication*, 标准的late fusion方法
>
> 3. *Multiplicative layer* 
>
>    [Learn to Combine Modalities in Multimodal Deep Learning](https://arxiv.org/abs/1805.11730)
>
> 4. *MMTM* 
>
>    [Mmtm: Multimodal transfer module for cnn fusion](https://openaccess.thecvf.com/content_CVPR_2020/html/Joze_MMTM_Multimodal_Transfer_Module_for_CNN_Fusion_CVPR_2020_paper.html)
>
> 5. *MSAF*:
>
>    [MSAF: Multimodal Split Attention Fusion](https://arxiv.org/abs/2012.07175)
>
> 6. *ERANNs*:
>
>    [ERANNs: Efficient Residual Audio Neural Networks for Audio Pattern Recognition](https://arxiv.org/abs/2106.01621)



