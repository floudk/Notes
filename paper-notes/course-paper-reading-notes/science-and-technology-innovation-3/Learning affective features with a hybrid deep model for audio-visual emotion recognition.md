### Introduction

This work proposes **a hybrid deep learning framework** composed by CNN, 3D-CNN, and DBN to learn a joint audio-visual feature representation for emotion classification.

This work uses a 2-phase model to do emotion recognition.

1. Pre-trained CNN and 3D-CNN models to lean audio and visual segment features respectively.
2. Combine the outputs from CNN and 3D-CNN models into a fusion network built with a DBN model to jointly learn a discriminative audio-visual segment feature representation.

### Background

1. Why recognizing human emotions with computer is still a challenging task?

   It is difficult to extract the best audio and visual features characterizing human emotions.

2. What is the key points in emotion recognition?

   *Feature extraction* and *multimodality fusion*

   In past years, a lot of hand-crafted features for emotion recognition has been proposed, but they all suffer from kind of emotional gap with human feelings. It is desirable to extract high-level features to effectively distinguish emotions. And after that, multimodality fusion is needed to integrate two modalities.

   ==Feature extraction==

   - **Audio** affect features can be categorized into three types:
     1. *Prosody* features, including pitch, intensity, energy, and duration time, are able to reflect *the rhythm of* spoken language.
     2. *Voice quality* features, including formants, spectral energy distribution.
     3. *Spectral features*, including Mel-frequency Cepstral Coefficient(*MFCC*)
   - **Visual** feature can be summarized into two categories:
     1. *static*: appearance-based feature extraction methods, which adopt the whole-face or specific regions in a face image to describe the subtle changes of the face such as wrinkles and furrows.
     2. *dynamic*: For dynamic image sequences, the deformations and facial muscle movements are considered.

   ==Multimodality Fusion==

   Typically, there can be four types of fusion:

   - **feature-level fusion**: Also called **early fusion**, the most common and straightforward way. The basic idea is that we can first extract features from single modal, and then concatenate the features into a giant feature vectors as the input of network. That's to say, we aim to build the map relation between the giant feature vector and the results.

     Defect: Feature-level fusion can not model the complicate relationships.

   - **decision-level fusion**: Also called **late fusion**, aims to combine several unimodal emotion recognition results through an algebraic combination rule, such as max, min, sum.

     Defect: Decision-level fusion can not capture the mutual correlation among different modalities since the assumption that modalities are independent.

   - **score-level fusion**: A special case of decision-level fusion(so it is also **late fusion**). Instead of using simple algebraic rule, this method weights independent decision outcomes. Each modal will give scores to different emotion results and is finally combined to indicate the likelihood of different emotion classes.

   - **model-level fusion**: Also called **model fusion**, a compromise between feature-level fusion and decision-level fusion. The basic idea is to obtain a joint feature representation of audio and visual modalities with a used fusion model.

   Some existed fusion work are mainly focusing on shallow feature fusion work.

3. What is the DBN network ?

   > DBNs are built by stacking multiple *Restricted Boltzmann Machines*([ref](https://zhuanlan.zhihu.com/p/34201655)), which is always used to *learn high-level feature representations* from low-level hand-crafted features.

4. Why is CNNs?

   CNN, consisting of convolutional layers and fully connected layers, can learn a *discriminative multi-level feature representation* from *raw inputs* and fully connected layers can be regarded as a *non-linear* classifier.
   
   Previous work (for the year 2017) mostly used a 2-D spatial CNN network to deal with images, which may ignore motions. However, 3D CNNs can compute feature maps from both spatial and temporal dimensions.
   
   > [A transformer based joint-encoding for emotion recognition and sentiment analysis](./A Transformer-based joint-encoding for Emotion Recognition and Sentiment Analysis.md) uses 2D-CNN for spatial and 1D-CNN for temporal analysis respectively

### Framework

<img src="imgs\Hybrid Deep Model Framework.png" alt="image-20220420220740044" style="zoom:200%;" />

The framework is comprised of three steps:

1. **Audio Feature**: Use Mel-spectrogram segment to present raw audio signals, which in the picture is shown like the RGB image, and input these data to a *pre-trained* deep CNN model.
2. **Video Feature**: The paper consider video features as *contiguous frames*, and input it to a *pre-trained* 3d-CNN network.
3. **Fusion Phase**: A fusion network built with *a deep DBN* is trained to predict correct emotion labels. The last output of DBN is just the audio-visual segment feature. Then, *an average-pooling* is employed to aggregate all segment features. Finally *a linear SVM* is used to classification.

> From my personal opinion, some paper[[1](./A cross-modal fusion network based on self-attention and residual structure for multimodal emotion recognition.md)] believes the method issued in this paper can be called as *late fusion*. Contrasted with [early fusion](Multi-modal emotion recognition on IEMOCAP with neural networks.md) work, late fusion methods tend to train network to get a joint high-level features, while early fusion do not train to get a joint feature but concatenate these features in some way (mostly kind of layers)

#### Inputs

Since video samples may have different durations, we split each of them into a certain number of overlapping segments, which is also kind of method to enlarge the dataset.

The whole log Mel-spectrogram from audio signals are first extracted, and then a fixed context window to split the spectrogram into these segments.

It is worthy mentioning that, the audio data is extracted to a 3D representations, which can be both beneficial to utilizing existed pre-trained CNN model, but also can do convolution along frequency and time axis.

### Experiments

#### Datasets

1. RML: The RML audio-visual dataset is composed of 720 video samples from 8 subjects, speaking the six different languages, including Mandarin, and containing 6 emotions.
2. eNTERFACE05: The eNTERFACE05 audio-visual acted dataset includes the six emotions, and only contains English speakers.
3. BAUM-1s: The dataset has the six basic emotions and some mental states, only in Turkish subjects.

## Highlights

1. Use pre-trained deep network to do first phase feature extraction.
2. Turn the 1D audio data to 3D RGB-like data to reuse pre-trained AlexNet in images. (kind of transfer learning)
3. Use DNB network to do fusion