## Description Based Text Classification with Reinforcement Learning

Authors: | _Duo Chai_ | _Wei Wu_ | _Qinghong Han_ | _Wu Fei_ | _Jiwei Li_

Paper Link: [here](https://arxiv.org/abs/2002.03067)

* * *

Note: This is a documentation of my understanding of the paper

* * *

### TL; DR / Summary

#### The Problem:

The standard methodology of text classification happens in two stages - _text feature extraction_ and _classification_. In this formulation categories are simply represented as indexes in the label vocabulary leaving the model with no explicit instructions on what to classify.

#### The Proposed Solution:

This paper suggests an alternative strategy wherein each category label is associated with a category description. These descriptions are generated using either hand-crafted examples or by using abstractive/extractive models using reinforcement learning. The methodology proposes a text entailment style approach where the text and description are concatenated and fed to a classifier to decide if the current label is to be assigned to the text.

#### Why this methodology?

This strategy forces the model to attend to the most salient (important) features of the text by baking in the label description (think of it like a hard version of attention) making it more information rich.

* * *

### Deeper analyses of the problem with the standard formulation

Let's first have a glimpse of BERT's attention mechanism. You can use [ExBERT](https://exbert.net/exBERT.html?model=bert-base-cased&modelKind=bidirectional&sentence=The%20girl%20ran%20to%20a%20local%20pub%20to%20escape%20the%20din%20of%20her%20city.&corpus=woz&layer=0&heads=..0,1,2,3,4,5,6,7,8,9,10,11&threshold=0.7&tokenInd=null&tokenSide=null&maskInds=..9&metaMatch=pos&metaMax=pos&displayInspector=null&offsetIdxs=..-1,0,1&hideClsSep=true) in order to visualize this or refer to the figures in [this paper](https://www-nlp.stanford.edu/pubs/clark2019what.pdf).

As you can see, the self-attention mechanism pays attention to only a handful of words in each of it's layers. This means that the actual class indicators in the text can be just a few keywords and could be deeply buried in the text making it hard to differentiate grain from chaff.

If you are dealing with a multi-label text classification problem, then signals from different classes might get entangled in the text as well. All this makes it difficult for the model as it needs to first learn to associate relevant text with target aspect and then decide the sentiment.

### Let's dive right in!

Past work on query generation:
1. Combine supervised learning + reinforcement learning to generate natural language descriptions
2. Generative model to generate queries based on unlabeled texts to train QA models
3. Framed the task of description generation as a seq2seq task, where descriptions are generated conditioning on the texts
4. Proposed a generator-evaluator framework that directly optimizes objectives
    - Generator - is a seq2seq model that incorporates the structure and semantics of the question being generated
    - Evaluator - evaluates and assigns a reward to each predicted question based on its conformity to the structure of ground-truth questions

This paper takes inspiration from 1. and 4. above and marries the two.

#### 
{[CLS]; q<sub>y</sub>; [SEP]; x} → transformers in BERT → contextual representation, h<sub>[CLS]</sub>