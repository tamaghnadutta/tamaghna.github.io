## Description Based Text Classification with Reinforcement Learning

Authors: | _Duo Chai_ | _Wei Wu_ | _Qinghong Han_ | _Wu Fei_ | _Jiwei Li_

Paper Link: [here](https://arxiv.org/abs/2002.03067)

* * *

### TL; DR / Summary

#### The Problem:

The standard methodology of text classification happens in two stages - _text feature extraction_ and _classification_. In this formulation categories are simply represented as indexes in the label vocabulary leaving the model with no explicit instructions on what to classify.

#### The Proposed Solution:

This paper suggests an alternative strategy wherein each category label is associated with a category description. These descriptions are generated using either hand-crafted examples or by using abstractive/extractive models using reinforcement learning. The methodology proposes a text entailment style approach where the text and description are concatenated and fed to a classifier to decide if the current label is to be assigned to the text.

#### Why this methodology?

This strategy forces the model to attend to the most salient (important) features of the text by baking in the label description (think of it like a hard version of attention) making it more information rich.
