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

### ??????

1. Emotion Recognition ??? Sentiment Analysis ?????????????????????

   Sentiment Analysis????????????????????????????????????????????????????????????

   Emotion Recognition????????????????????????????????????????????????????????????

2. Attention???????????????????????? [???Attention??????????????????????????????????????????](https://zhuanlan.zhihu.com/p/362366192)

   Attention??????????????????????????????????????????????????????????????????????????????

   - ????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????Fancy.

   - Attention?????????????????????

     $Attention(Q,K,V)=softmax(\frac {QK^T}{\sqrt d_k})V$

     ??????????????????*????????????????????????????????????*???

   - Transformer: Attention is all you need

     ??????recurrent???Attention??????????????????????????????????????????????????????????????????????????????????????????????????????????????????convolution?????????????????????????????????

   - BERT: Attention?????????

     BERT???????????????????????????????????????????????????????????????????????????????????????fine-tune??????????????????????????????????????????????????????????????????

   - CV: Attention???????????????????????????????????????

     attention????????????????????????????????????????????????????????????????????????

     ?????????CNN???Attention?????????????????????????????????CNN?????????????????????????????????????????????

   - CV: Attention will be all you need.

     ???????????????transformer???????????????????????????? 

     *Vision Transformer*(Google) ??????????????????????????????????????????Transformer?????????????????????????????????

   - ??????????????????Attention?????????GNN?????????

     ??????Attention?????????????????????????????????

   - ?????????????????????Attention?????????

     Attention???????????????????????????????????????????????????????????????????????????????????????????????????????????????

   - Attention?????????????????????????????????

     attention????????????????????????????????????????????????????????????????????????????????????

3. Transformer????????????????????????????

   Transformer ??????????????????*??????-?????????*, ??????????????????CNN???RNN??????????????????*Attention*???????????????

   - *Encoder*:?????????????????????????????? ?????????????????????????????????????????????

     Transformer??????RNN????????????????????????????????????????????????????????????????????????

     self-attention

     ???????????????????????????

     ????????????

   - *Decoder*: ??????????????????????????????????????????????????????

4. MHA???????????????????????????*????????????*????????????

   *???????????????*??????????????????Q, K, V??????????????????????????????????????????output????????????????????????output???????????????????????????output???

   ???????????????query???key???value?????????????????????????????????????????????*????????????*???