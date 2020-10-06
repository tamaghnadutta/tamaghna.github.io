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

#### How is the description-based text classification formulated?

You concatenate the query, **q<sub>y</sub>** with the text, **x** and feed it to a transformer and get the **h<sub>[CLS]</sub>** which encodes the entire query + text. (The **[CLS]** token bakes in all the features from the input text and is what is sent forward to the softmax/sigmoid layer for classification)

{**[CLS]**; q<sub>y</sub>; **[SEP]**; x} → transformers in BERT → contextual representation, **h<sub>[CLS]</sub>**

You then pass it through the sigmoid layer,

p(y\|x) = sigmoid(**W<sub>2</sub>**ReLU(**W<sub>1</sub>**h<sub>[CLS]</sub> + **b<sub>1</sub>**) + **b<sub>2</sub>**) → value between **0** and **1**

For single-label classification, you just take the argmax of the sigmoid (which is nothing but the softmax)

y˜ = **arg max**<sub>y</sub>({p(y\|x), **∀y ∈ Y**})

For multi-label classification,

y˜ = {y \| p(y\|x) > 0.5, **∀y ∈ Y**}

These are binary classifiers and you can have N-binary classifiers like so.

You may also formulate your multi-class problem as a single N-class classifier by concatenating all descriptions with the input **x**,

{**[CLS<sub>1</sub>]**; q<sub>1</sub>; **[CLS<sub>2</sub>]**; q<sub>2</sub>; ...; **[CLS<sub>N</sub>]**; q<sub>N</sub> ; **[SEP]**; x} → fed to transformer → h<sub>[CLS<sub>1</sub>]</sub>, h<sub>[CLS<sub>2</sub>]</sub>, ..., h<sub>[CLS<sub>N</sub>]</sub>

Do **note** though that the N-class classifier strategy cannot handle the multi-label classification case.

The probability of assigning class _n_ to instance _x_ is obtained by first mapping h<sub>[CLS<sub>n</sub>]</sub> to scalars and then applying a softmax on it

a<sub>n</sub> = h<sup>^</sup><sup>T</sup>.h<sub>[CLS<sub>n</sub>]</sub>

p(y=n\|x) = softmax(a<sub>n</sub>)

#### What are the ways to construct description?

There are primarily three strategies:
- Template strategy
- Abstractive Strategy
- Extractive Strategy

Important thing to note here is that the goal is to have the ability for the model to generate the most _appropriate descriptions_ of the different classes _conditioned on the current text to classify_, and the appropriateness of the generated descriptions should _directly correlate_ with the _final classification performance_.

The Template Strategy is usually very labour intensive and also human generated templates might be sub-optimal. So let's have a deeper look at the Extractive and Abstractive strategies.

If you have worked on any text summarisation problem, then you can build your intuition around the Extractive Strategy by looking at SQuAD style text extraction from the body of the text and you can build your intuition around the Abstractive Strategy by looking at GPT-2/GPT-3 style generative text generation.

#### Description Construction : Extractive Strategy

For each input, x = {x<sub>1</sub>, · · · , x<sub>T</sub>} the extractive model generates a description **q<sub>yx</sub>** for each class label **y**; where **q<sub>yx</sub>** is a _substring_ of **x**. For different inputs **x**, descriptions for the same class can be different. This is quite intuitive as when **x** changes, the base text on which extractions are done changes and thus the descriptions also change.

It's important to note that -
* For the _golden class_, **y** that should be assigned to **x**, there should always be a substring of **x** relevant to **y**.
* But for classes that should not be assigned, there might _not be_ corresponding substrings in **x** that can be used as descriptions.

To deal with this problem, _append N dummy tokens_ to **x** such that if the extractive model picks a dummy token it will fall back to using hand-crafted templates for different categories as descriptions. Also, note that in this case, the hand-crafted examples are crafted by using some regex heuristics rather than completely manual.

Now you must have been wondering, when does the reinforcement learning bit come in and how does it actually help?

Reinforcement learning is used to back-propagate the signal indicating which span contributes how much to the classification performance. Let's also introduce some typical reinforcement learning components here -

* action, **a**
* policy, **π**
* reward, **r**

##### Action and Policy

For each class label **y**, ***action*** is to pick a text-span {x<sub>i<sub>s</sub></sub> , · · · , x<sub>i<sub>e</sub></sub>} from **x** to represent **q<sub>yx</sub>** → need start and end index of span, **a<sub>i<sub>s</sub>,i<sub>e</sub></sub>**

For each class label **y**, the ***policy*** **π** defines the probability of selecting the start index, i<sub>s</sub> and end index, i<sub>e</sub>.

Each token x<sub>k</sub> within **x** is mapped to a representation h<sub>k</sub> using BERT.

P<sub>start</sub>(y, k) = exp(**W<sub>ys</sub>**h<sub>k</sub>)/(Σ<sub>1...T</sub> exp(**W<sub>ys</sub>**h<sub>t</sub>))

P<sub>end</sub>(y, k) = exp(**W<sub>ye</sub>**h<sub>k</sub>)/(Σ<sub>1...T</sub> exp(**W<sub>ye</sub>**h<sub>t</sub>))

Each class **y** has a class-specific **W<sub>ys</sub>** and **W<sub>ye</sub>**

Probability of a text span with the starting index i<sub>s</sub> and ending index i<sub>e</sub> being the description for class **y**, P<sub>span</sub>(y, a<sub>is,ie</sub>) is -

**P<sub>span</sub>(y, a<sub>is,ie</sub>) = P<sub>start</sub>(y, i<sub>s</sub>) × P<sub>end</sub>(y, i<sub>e</sub>)**

##### Reward

Given **x** and the description **q<sub>yx</sub>**, classification model will assign probability of assigning correct label to **x** which will be used as ***reward*** to update both the classification model and the extractive model.

For multi-class classification, ***reward*** is given by -

R(x, q<sub>yx</sub> for all **y**) = p(y = n\|x) where, **n** is gold label for **x**

#### REINFORCE

To find optimal policy, use **REINFORCE** algorithm which maximizes the expected reward E<sub>π</sub>[R(x, q<sub>y</sub>)]

For each generated description **q<sub>yx</sub>** and the corresponding **x**,

**_L_ = −E<sub>π</sub>[R(q<sub>yx</sub>, x)]**

**REINFORCE** approximates above equation with sampled distributions from the policy(π) distribution and the gradient to update parameters is given by -

**∇_L_ ≈ -Σ<sub>i=1...B</sub>∇logπ(a<sub>i<sub>s</sub>,i<sub>e</sub></sub>\|x, y)[R(q<sub>y</sub>)-b]**

where, **b** denotes the baseline value, which is set to the average of all previous rewards.

So, the Extractive Policy is initialized to generate sub-text as descriptions. Then the extractive model and the classification model are jointly trained based on the reward.