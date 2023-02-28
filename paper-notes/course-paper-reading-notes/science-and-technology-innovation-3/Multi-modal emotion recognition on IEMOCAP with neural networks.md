### Introduction

This is a *early fusion* way to perform multimodal emotion recognition on *IEMOCAP* dataset.

> **IEMOCAP**: one of the biggest open-sourced multimodal resources available in emotion detection area, consisting of approximately 12 hours of audio-visual data, including facial recording, speech and text transcriptions.

> **Early Fusion**: As the name implies, *early fusion* is carried out at feature level, that is to say, various features are concatenated into one supervector for classification; 
>
> while the *late fusion* methods fuse the modalities in classification scores level by using supervised/semi-supervised learners directly.

This model is straightforward in fusing modals. It firstly explore the best individual detection accuracy from each of the different modes, and then combine them in an *ensemble based architecture*, which consists of *Long Short Term Memory networks*, *CNN*, *fully connected Multi-Layer Perceptrons*, *Dropout*, *adaptive optimizers*, *pretrained word-embedding models* and *Attention based RNN decoders*.

As far as the writers concerned, the main benefits could be:

1. Early fusion makes the whole model be modular. We can modify or update one of the models and do not effect other models at all.
2. Use Motion-capture data instead of Video recording, hence a lot of computation resources and memory resources are reduced.
3. This task is **open-sourced**. [github](https://github.com/Samarth-Tripathi/IEMOCAP-Emotion-Detection)

### Related Works

1. Early work on *IEMOCAP* are focus on speech instead of Multi-modal.
2. State-of-art classification on *IEMOCAP* is [Multimodal Sentiment Analysis](https://ieeexplore.ieee.org/abstract/document/8636432), which uses 3D-CNN for visual feature extraction, text-CNN for textual features extraction and openSMILE for audio feature extraction.  That's to say, it is also kind of *early-fusion* method.

### Experimental Setup

In *IEMOCAP*, there are 10 classifications of emotions: neutral, happiness, sadness, anger, surprise, fear, disgust frustration, excited and other.

In this paper, only 4 of them are considered: anger, excitement, neutral and sadness. And three models are used here: *Speech, Text, Mocap*.

- *Speech*: For speech, the **MFCC** is used in this paper.
- *Text*: **pre-trained Glove embeddings** is used in this paper.
- *Mocap*: This paper do not use video streaming data, and do a sample to all the features in *Mocap* between the start and finish time values.

### Models

Basically speaking, this paper uses several models to do *speech recognition*, *text recognition* and *Mocap recognition*. After selecting the best models from them, we take these models for each modality without their final SoftMax layers, and perform feature fusion by concatenating their final fully connected layers and other optimizations.



