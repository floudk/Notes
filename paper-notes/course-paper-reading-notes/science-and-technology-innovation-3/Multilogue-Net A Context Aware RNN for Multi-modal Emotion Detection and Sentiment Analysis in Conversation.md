### Introduction

*Multilogue-Net*: An end to end RNN architecture that attempts to take into account all the mentioned drawbacks, assuming that the emotion predominantly depends on 4 factors:

- interlocutor state
- interlocutor intent(aim): hard to model due to its dependency of prior knowledge about the speaker.
- the preceding and future emotions
- the context of the conversation

### Background

1. What is a *pairwise attention mechanism*?

2. What is the *gated recurrent units*(GRU)?

   LSTM and GRU were created as the solution to short-term memory, and GRU is kind of simplification of LSTM.
   <img src="imgs\GRU.png" alt="image-20220424204846401" style="zoom:40%;" />

3. What is a softmax() function ?

   To understand the *softmax*, it is worth mentioning *hardmax*. Hardmax is just a max function. However, hardmax is not suitable for learning. Since lack of normalization and smoothness, we need a softmax function.

   $Softmax(z_i)=\frac{e^{z_i}}{\sum_{c=1}^C e^{z_C}}$

### Model Details

#### Problem Formulation

- A conversation consists of P number of participants: $p_1,p_2,...p_P$.
- Each utterance, $u_1,u_2,...n_N$ corresponds to a particular participant of the conversation.
- Each utterance $u_t(p)$, where p is the party who uttered the utterance, there exist three independent representations $t_t\ \ a_t \ \ v_t$,  which are obtained using the **feature extractors**.

Hence, the problem is build a map. The *input* includes three independent data:

1. three independent representations of a particular utterance: text, audio and video
2. information regarding the previous emotional state of the participant
3. a representation of the current context of the conversation.

and the output prediction of a sentiment score and emotion label.

#### Model

The basic assumption is that: the sentiment or emotion of an utterance predominantly depends on four factors as mentioned before:

1. Interlocutor State:  Interlocutor state is modelled using a state GRU(**sGRU**)
2. Interlocutor Intent
3. Context of the conversation until that point: A context GRU(**cGRU**) is used to keep track of the context of the conversation.
4. Previous interlocutor states and emotions of a particular participant in the conversation: an emotion GRU(**eGRU**) is used to keep track of the emotional state of that particular participant.

However, like mentioned before, the interlocutor intent needs additional information to model, so the proposed model only model other three explicitly, and assume that the intent can be modelled implicitly during training.

![image-20220424202424313](imgs\Multilogue.png)

Finally, a pairwise attention mechanism will use all modality outputs(i.e., combine or just use alone) at a particular timestamp to predict emotion at that timestamp.

*To sum up*:

There are three modalities: text, audio and video. They will be treated independently until pairwise attention phase.
The model provide one sGRU and one eGRU for every modality and participant, and a cGRU for each modality common to all participants in the conversation.
The current input(including context, state) and previous(i.e., at last timestamp) state, context, and emotion representations are taken as input to predict current emotions.

#### cGRU

Each modality owns one cGRU to capture the context of the conversation by jointly encoding the utterance representation of that modality and the previous timestamp speaker state GRU output of that modality.

$c_{t+1}^t=cGRU(c_t^t,(t_t\oplus s_t^t))$

$c_{t+1}^a=cGRU(c_t^a,(a_t\oplus s_t^a))$

$c_{t+1}^v=cGRU(c_t^v,(v_t\oplus s_t^v))$

Here, the t, a and v represent three modalities respectively.

- $c_{t+1}^a$ : the context of a modal in next timestamp.
- $c_t^a$: the context of a modal in current timestamp.
- $a_t$: the input of current modal.
- $s_t^a$: the state of current speaker
- $\oplus$: concatenation

#### sGRU

For a conversation model with m modalities and p participants, we need p*m number of sGRU.

The output of a sGRU associated with a participant is a fixed size vectors which serve as an encoding to represent the interlocutor state, which is used directly used for *both prediction and context updating*.

The state update of a certain person in one modality is using the *input feature representation* of that modality and *simple attention over all the context vectors* until that timestamp. The simple attention mechanism can be described as:

$\alpha = softmax(m_t^T W_\alpha [c_1^m,c_2^m,...c_t^m])$
$att_t = \alpha [c_1^m,c_2^m,...c_t^m,]^T$

$m_t^T$ is the traverse of one of modalities input; 
$W_\alpha$ is the weight matrix, in $R^{D_{modal}\times D_c}$, $D_{modal}$ is the size of utterance representation of certain modal and $D_c$ is the size of the context vector.
$att_t\in R^{D_c}$.

After get $att_t$ matrixes, we then update state.

$s_{t+1}^t=sGRU(s_t^t,(t_t\oplus att_{t+1}^t))$

$s_{t+1}^a=sGRU(s_t^a,(a_t\oplus att_{t+1}^a))$

$s_{t+1}^v=sGRU(s_t^v,(v_t\oplus att_{t+1}^v))$

The output of the sGRU for modality *m* and timestamp *t* serves as *an encoding of the speaker state*
as conveyed by modality m, at time t  

#### eGRU

The emotion GRU serves as the decoder for the encoding produced by the state GRU and produce an emotion or sentiment representation which is further used by the pairwise attention mechanism.

$e_{t+1}^t=eGRU(e_t^t,s_{t+1}^t)$

$e_{t+1}^a=eGRU(e_t^a,s_{t+1}^a)$

$e_{t+1}^v=eGRU(e_t^v,e_{t+1}^v)$

The emotion GRU acts as a decoder to the encoding produced by the associated state GRU, producing a vector which can be used for both sentiment and emotion prediction.

#### Pairwise Attention Mechanism

The eGRU will produce one vector for each modality in each timestamp t.

**Pairwise attention** is then used over these m vectors to produce the final prediction output.   

In this particular dataset, the pairwise could be:

- $(e^v,e^t)$
- $(e^t,e^a)$
- $(e^a,e^v)$

Finally, the model further concatenate these outputs to make the final prediction.

#### Final Predictions

The prediction layer *varies* based on whether a sentiment or emotion prediction is expected.  

- sentiment prediction

  sentiment prediction will output a value between -1 and +1 at timestamp t

- emotion prediction

  emotion prediction will calculate 6 emotion class probabilities from $L_t$

### Interesting Analysis

1. The fusion mechanism(i.e., pairwise attention) works only when text modality is involved.
2. eGRU is essential for the task.

