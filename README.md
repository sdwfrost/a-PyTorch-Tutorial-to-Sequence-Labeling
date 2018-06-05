This is a **[PyTorch](https://pytorch.org) Tutorial to Sequence Tagging**.

This is the second in a series of tutorials I plan to write about _implementing_ cool models on your own with the amazing PyTorch library.

Basic knowledge of PyTorch, recurrent neural networks is assumed.

Questions, suggestions, or corrections can be posted as issues.

I'm using `PyTorch 0.4` in `Python 3.6`.

# Contents

[***Objective***](https://github.com/sgrvinod/caption#objective)

[***Concepts***](https://github.com/sgrvinod/caption#concepts)

[***Overview***](https://github.com/sgrvinod/caption#overview)

[***Implementation***](https://github.com/sgrvinod/caption#implementation)

[***Frequently Asked Questions***](https://github.com/sgrvinod/caption#faqs)

# Objective

**To build a model that can tag each word in a sentence with entities, parts of speech, etc.**

Let's implement the [_Empower Sequence Labeling with Task-Aware Neural Language Model_](https://arxiv.org/abs/1709.04109) paper. This is more complex than most sequence tagging models, but you will learn many useful concepts - and it's pretty powerful.

This model augments the sequence labeling task by training it _concurrently_ with language models.

Here are some examples of this model tagging entities on sentences not seen during training or validation:

# Concepts

* **Sequence Labeling**

* **Language Models**

* **Co-Training**

* **Subword Features with Character RNNs**

* **Conditional Random Fields**

* **Viterbi Decoding**

* **Highway Networks**

# Overview

In this section, I will present a broad overview of this model. If you're already familiar with it, you can skip straight to the implementation.

### LM-LSTM-CRF

The authors refer to the model as the "Language Model - Long Short-Term Memory - Conditional Random Field" since it involves co-training language models with an LSTM (RNN) + CRF combination.

![](./img/framework.png)

This image from the paper thoroughly represents the whole model, but don't worry if it seems too complex at this time. We'll break it down and take a closer look at the components.

### Co-training

**Co-training is when you simultaneously train a model on two or more tasks.**

Usually we're only interested in _one_ of these tasks - in this case, the sequence labeling.

But when layers in a neural network contribute towards performing multiple functions, they learn more than they would have if they had trained only on the primary task. This is because the information extracted at each layer is expanded to accomodate all tasks. When there is more information to work with, **performance on the primary task is enhanced**.

Enriching existing features in this manner removes the need for using handcrafted features for sequence labeling.

The **total loss** during co-training is usually a linear combination of the losses on the individual tasks. The parameters of the combination can be fixed or learned as updateable weights.

<p align="center">
![](./img/loss.png)
</p>

Since we're aggregating individual losses, you can see how upstream layers shared by multiple tasks would receive updates from all of them during backpropagation.

<p align="center">
![](./img/update.png)
</p>

The authors of the paper **simply add the losses** (`β=1`), and we will do the same.

Let's take a look at the tasks that make our model. **There are _three_**.

![](./img/forwchar.jpg)

This leverages sub-word information to predict the next word.

We do the same in the backward direction.
![](./img/backchar.jpg)

We also use the outputs of these two Character RNNs as inputs to our Word RNN and Conditional Random Field to perform our primary task of sequence labeling.
![](./img/word.jpg)

We're using sub-word information for our tagging task because it can be a powerful indicator of the tags, whether they're parts of speech or entities. For example, ir may learn that adjectives commonly end with "-y" or "-ul", places  often end with "-land" or "-burg".

But our sub-word features, i.e. the output of the Character RNNs, is also enriched with additional information - the knowledge it needs to predict the next word in both forward and backward directions, because of models 1 and 2.

Therefore, our sequence tagging model uses both
- word-level information (word embeddings).
- character-level information up to and including this word in both directions, enriched with the know-how required to be able to predict the next word in both directions.

The Bidirectional LSTM/RNN encodes these features into new features at each word containing information about the word and its neighborhood, at both the word-level and the character-level. This forms the input to the Conditional Random Field.

### Conditional Random Field (CRF)

Without a CRF, we could have simply used a single linear layer to transform the output of the Bidirectional LSTM into scores for each tag. These are known as **emission scores**, which are a representation of the likelihood of the word being a certain tag.

A CRF calculates not only the emission scores but also the **transition scores**, which are the likelihood of a word being a certain tag _considering_ the previous word was a certain tag. Therefore the transition scores measure how likely it is to transition from one tag to another.

If there are `m` tags, transition scores are stored in a matrix of dimesions `m, m`, where the rows represent the tag of the previous word and the columns represent the tag of the current word. A value in this matrix at position `i, j` is the likelihood of transitioning from the `i`th tag at the previous word to the `j`th tag at the current word. Unlike emission scores, transition scores are not defined for each word in the sentence. They are global.

In our model, the CRF layer outputs the **aggregate of the emission and transition scores at each word**.

For a sentence of length `L`, emission scores would be an `L, m` tensor. Since the emission scores at each word do not depend on the tag of the previous word**, we create a new dimension like `L, _, m` and broadcast (copy) the tensor along this direction to get an `L, m, m` tensor.

The transition scores are an `m, m` tensor. Since the transition scores are global and do not depend on the word, we create a new dimension like `_, m, m` and broadcast (copy) the tensor along this direction to get an `L, m, m` tensor.

We can now **add them to get the total scores which are an `L, m, m` tensor**. A value at position `k, i, j` is the aggregate of the emission score of the `k`th word for the `j`th tag _and_ the transition score of the `k`th word being the `j`th tag considering the previous word was the `i`th tag.

For our example sentence `dunston checks in <end>`, if we assume there are 5 tags in total, the total scores would look like this -

![](./img/tagscores0.jpg)

But wait a minute, why are there `<start>` end `<end>` tags? While we're at it, why are we using an `<end>` token?

### About `<start>` and `<end>` tags, `<start>` and `<end>` tokens

Since we're modeling the transitioning between tags, we also include a `<start>` tag and an `<end>` tag in our tag-set.

The transition score of a certain tag given that the previous tag was a `<start>` tag represents the **likelihood of this tag being the _first_ tag in a sentence**. For example, sentences usually start with articles (a, an, the) or nouns or pronouns.

The transition score of the `<end>` tag considering a certain previous tag indicates the **likelihood of this previous tag being the _last_ tag in a sentence**.

We will use an `<end>` token in all sentences and not a `<start>` token because the total scores at each word are defined with respect to the _previous_ word's tag, which would make no sense for a `<start>` token.

The correct tag of the `<end>` token is always the `<end>` tag. The "previous tag" of the first word is always the `<start>` tag.

To illustrate, if our example sentence `dunston checks in <end>` had the tags `tag_2, tag_3, tag_3, <end>`, the values in red indicate the scores of these tags.

![](./img/tagscores.jpg)

Remember that **this is just convention.** You could just as well define the output of the CRF to be scores `current_tag -> next_tag` instead of `previous_tag -> current_tag`. In this case you would broadcast the emission scores like `L, m, _`, and you would have a `<start>` token in every sentence instead of an `<end>` token. The correct tag of the `<start>` token would always be the `<start>` tag. The "next tag" of the last word would always be the `<end>` tag.

### Highway Networks

We generally use activated linear layers to transform and process outputs of an RNN/LSTM.

If you're familiar with residual connections, we can add the input before the transformation to the transformed output, creating a path for data-flow around the transformation.

<p align="center">
![](./img/nothighway.png)
</p>

This path is a shortcut for the flow of gradients during backpropagation, and aids in the convergence of deep networks.

A highway network is similar to a residual network, but we use a sigmoid-activated gate to determine the ratio in which the input and transformed output is combined.

<p align="center">
![](./img/highway.png)
</p>

The authors of the paper make the case that using highway networks instead of regular linear networks improves performance.

Highway networks are used at three points in our combined model -

- to transform the output of the forward Character RNN to predict the next word
- to transform the output of the backward Character RNN to predict the next word (in the backward direction)
- to transform the concatenated output of the forward and backward Character RNNs for use in the word-level RNN along with the word embedding.

Since the Character RNNs contribute towards multiple tasks during co-training, highway networks are beneficial in extracting task-specific information from its outputs.

### Putting it all together

It might be clear by now what our combined network looks like.

![](./img/model.jpg)

### Other configurations

Progressively removing parts of our network results in progressively simpler networks that are used widely for sequence labeling.

![](./img/configs.jpg)

#### (a) a Bi-LSTM + CRF sequence tagger that leverages sub-word information.

There is no co-training.

Using character-level information without co-training still improves performance.

#### (b) a Bi-LSTM + CRF sequence tagger.

There is no co-training or character-level processing.

This configuration is used quite commonly in the industry and works well.

#### (c) a Bi-LSTM sequence tagger.

There is no co-training, character-level processing, or a CRF. Note that a linear or highway layer would replace the latter.

This could work reasonably well, but a Conditional Random Field provides a sizeable performance boost.

### Viterbi Loss

Since we're not using a linear layer that computes only the emission scores, Cross Entropy is not a suitable loss metric.

Instead we will use the **Viterbi Loss** which, like the Cross Entropy, is a "negative log likelihood". But here you will be measuring the likelihood of the gold (true) tag sequence, instead of the likelihood of the true tag at each word in the sequence. To find the likelihood, you can take the softmax over the scores of all tag sequences.

The score of a tag sequence `t` is defined as the sum of the scores of the individual tags considering the tag of the previous words.

<p align="center">
![](./img/vscore.png)
</p>

(For example, consider the CRF scores we looked at earlier. The score of the tag sequence `tag_2, tag_3, tag_3, <end> tag` is the sum of the values in red, `4.85 + 4.79 + 3.85 + 3.52 = 17.01`.)

The Viterbi Loss is then defined as

<p align="center">
![](./img/vloss1.png)
</p>

where the gold tag sequence is `t_G` and the space of all possible tag sequences is `t`.

This simplifies to

<p align="center">
![](./img/vloss3.png)
</p>

Therefore, the Viterbi Loss is the difference between the log-sum-exp of the scores of all possible tag sequences and the score of the gold tag sequence, i.e. `log-sum-exp(all scores) - gold score`.

### Viterbi Decoding

Viterbi Decoding is a way to construct the most optimal tag sequence, considering not only the likelihood of a tag at a certain word (emission scores), but also the likelihood of a tag considering the previous and next tags (transition scores).

Once you generate CRF scores in a `L, m, m` matrix for a sequence of length `L`, we start decoding.

Let's decode the example CRF scores we looked at earlier.

![](./img/tagscores0.jpg)

For the first word in the sequence, the `previous_tag` can only be `<start>`. Therefore, only consider that one row.

These are also the cumulative scores for each `current_tag` at the first word.

![](./img/dunston.jpg)

Also keep track of the `previous_tag` that corresponds to each score. These are known as _backpointers_. At the first word, they are obviously all `<start>` tags.

At the second word, add the previous cumulative scores to the CRF scores of this word to generate new cumulative scores.

Note that the first word's `current_tag`s are the second word's `previous_tag`s. Therefore, broadcast the first word's cumulative score along the `current_tag` dimension.

![](./img/checks1.jpg)

For each `current_tag`, consider only the maximum of the scores from all `previous_tag`s.

Store backpointers, i.e. the previous tags that correspond to these maximum scores.

![](./img/checks2.jpg)

Repeat this process at the third word.

![](./img/in1.jpg)
![](./img/in2.jpg)

...and the last word, which is the `<end>` token.

Here, the only difference is you _ already know_ the correct tag. You need to store the maximum score and backpointer only for the `<end>` tag.

![](./img/end1.jpg)
![](./img/end2.jpg)

Now that you accumulated CRF scores across the entire sequence, **you trace _backwards_ to reveal the tag sequence with the highest possible score**.

![](./img/backtrace.jpg)

Therefore, the the most optimal tag sequence for `dunston checks in <end>` is `tag_2 tag_3 tag_3 <end>`.

# Implementation

The sections below briefly describe the implementation.

They are meant to provide some context, but **details are best understood directly from the code**, which is quite heavily commented.

### Dataset

I use the CoNLL 2003 NER dataset to compare my results with the paper.

Here's a snippet -

```
-DOCSTART- -X- O O

EU NNP I-NP I-ORG
rejects VBZ I-VP O
German JJ I-NP I-MISC
call NN I-NP O
to TO I-VP O
boycott VB I-VP O
British JJ I-NP I-MISC
lamb NN I-NP O
. . O O
```

This dataset is not meant to be publicly distributed, although you may find it somewhere online.

You will find several public datasets online that you can use to train the model. These may not all be 100% human annotated, but they are sufficient.

For NER tagging, you can use the [Groningen Meaning Bank](http://gmb.let.rug.nl/data.php).

For POS tagging, NLTK has a small dataset available you can access with `nltk.corpus.treebank.tagged_sents()`.

You would either have to reformat it to the CoNLL 2003 NER data format, or modify the code referenced in the **Data Pipeline** section.

### Inputs to model

We will need eight inputs.

#### Words

These are the word sequences that must be tagged.

`dunston checks in`

As discussed earlier, we will not use `<start>` tokens but we will need to use `<end>` tokens.

`dunston, checks, in, <end>`

Since we pass the sentences around as fixed size Tensors, we need to pad sentences (which are naturally of varying length) to the same length with `<pad>` tokens.

`dunston, checks, in, <end>, <pad>, <pad>, <pad>, ...`

Furthermore, we create a `word_map` which is an index mapping for each word in the corpus, including the `<end>`, and `<pad>` tokens. PyTorch, like other libraries, needs words encoded as indices to look up embeddings for them or to identify their place in the predicted word scores.

`4381, 448, 185, 4669, 0, 0, 0, ...`

The **word sequences fed to the model must be an `Int` tensor of dimensions `N, L_w`** where `N` is the batch_size and `L_w` is the padded length of the word sequences (usually the length of the longest word sequence).

#### Characters (Forward)

These are the character sequences in the forward direction.

`'d', 'u', 'n', 's', 't', 'o', 'n', ' ', 'c', 'h', 'e', 'c', 'k', 's', ' ', 'i', 'n', ' '`

We need `<end>` tokens in the character sequences to match the `<end>` token in the word sequences. Since we're going to use character level features at each word in the word sequence, we would also need character level features at `<end>` in the word sequence.

`'d', 'u', 'n', 's', 't', 'o', 'n', ' ', 'c', 'h', 'e', 'c', 'k', 's', ' ', 'i', 'n', ' ', <end>`

We would also need to pad them.

`'d', 'u', 'n', 's', 't', 'o', 'n', ' ', 'c', 'h', 'e', 'c', 'k', 's', ' ', 'i', 'n', ' ', <end>, <pad>, <pad>, <pad>, ...`

And encode them with a `char_map`.

`29, 2, 12, 8, 7, 14, 12, 3, 6, 18, 1, 6, 21, 8, 3, 17, 12, 3, 60, 0, 0, 0, ...`

The **forward character sequences fed to the model must be an `Int` tensor of dimensions `N, L_c`**, where `L_c` is the padded length of the character sequences (usually the length of the longest character sequence).

#### Characters (Backward)

This would be processed the same as the forward sequence, but backward. The `<end>` tokens still occur at the end though.

`'n', 'i', ' ', 's', 'k', 'c', 'e', 'h', 'c', ' ', 'n', 'o', 't', 's', 'n', 'u', 'd', ' ', <end>, <pad>, <pad>, <pad>, ...`

`12, 17, 3, 8, 21, 6, 1, 18, 6, 3, 12, 14, 7, 8, 12, 2, 29, 3, 60, 0, 0, 0, ...`

The **backward character sequences fed to the model must be an `Int` tensor of dimensions `N, L_c`**.

#### Character Markers (Forward)

These markers are positions in the character sequences where we extract features to either
- generate the next word in the language models, or,
- use as character-level features in the word-level RNN in the sequence labeler

Therefore, we must extract features at the end of every space `' '` in the character sequence, and also at the `<end>` token. We extract these indices for the forward character sequence.

`7, 14, 17, 18`

These are points after `dunston`, `checks`, `in`, `<end>` respectively. We have a marker for each word in the word sequence, which makes sense. (In the language models, however, since we're predicting the _next_ word, we won't predict at the marker which corresponds to `<end>`.)

We also pad these with `0`s. It doesn't matter what we pad with as long as they're valid indices. We will extract features at the pads, but we will not use them.

`7, 14, 17, 18, 0, 0, 0, ...`

We pad them to the padded length of the word sequences, `L_w`.

The **forward character markers fed to the model must be an `Int` tensor of dimensions `N, L_w`**.

#### Character Markers (Backward)

For the markers in the backward character sequences, we similarly find the positions of every space `' '` and the `<end>` token.

We also ensure that these **positions are in the same word order as in the forward markers**. This alignment makes it easier to concatenate features extracted from the forward and backward character sequences, and also prevents having to re-order the targets in the language models.

`17, 9, 2, 18`

These are points after `notsnud`, `skcehc`, `ni`, `<end>` respectively.

We pad with `0`s.

`17, 9, 2, 18, 0, 0, 0, ...`

The **backward character markers fed to the model must be an `Int` tensor of dimensions `N, L_w`**.

#### Tags

Let's assume the correct tags for `dunston, checks, in, <end>` are

`tag_2, tag_3, tag_3, <end>`

We have a `tag_map` (containing the tags `<start>`, `tag_1`, `tag_2`, `tag_3`,  `<end>`).

Normally, we would just encode them directly (before padding).

`2, 3, 3, 5`

These are `1D` encodings, i.e., tag positions in a `1D` tag map.

But the output of the CRF layer are `2D` `m, m` tensors at each word. We would need to encode tag positions in these `2D` outputs.

![](./img/tagscores.jpg)

The correct tag positions are marked in red.

`(0, 2), (2, 3), (3, 3), (3, 4)`

If we unroll these scores into a `1D` `m*m` tensor, then the tag positions in the unrolled tensor would be

```python
tag_map[previous_tag] * len(tag_map) + tag_map[current_tag]
```

Therefore, we encode `tag_2, tag_3, tag_3, <end>` as

`2, 13, 18, 19`

Note that you can retrieve the original `tag_map` indices by taking the modulus

```python
t % len(tag_map)
```

This will also be padded to the padded length of the word sequences, `L_w`.

The **tags fed to the model must be an `Int` tensor of dimensions `N, L_w`**.

#### Word Lengths

These are the lengths of the word sequences including the `<end>` tokens. Since PyTorch supports dynamic graphs, we will compute only over these lengths and not over the `<pads>`.

The **word lengths fed to the model must be an `Int` tensor of dimensions `N`**.

#### Character Lengths

These are the lengths of the character sequences including the `<end>` tokens. Since PyTorch supports  dynamic graphs, we will compute only over these lengths and not over the `<pads>`.

The **character lengths fed to the model must be an `Int` tensor of dimensions `N`**.

### Data pipeline

See `read_words_tags()` in `utils.py`.

This reads the input files in the CoNLL 2003 format, and extracts the word and tag sequences.

See `create_maps()` in `utils.py`.

Here, we create encoding maps for words, characters, and tags. We bin rare words and characters as `<unk>`s.

See `create_input_tensors()` in `utils.py`.

We generate the eight inputs detailed in the **Inputs to Model** section.

See `load_embeddings()` in `utils.py`.

We load pre-trained embeddings, with the option to expand the `word_map` to include out-of-corpus words present in the embedding vocabulary. Note that this may also include rare in-corpus words that were binned as `<unk>`s earlier.

See `WCDataset` in `datasets.py`.

This is a subclass of PyTorch [`Dataset`](https://pytorch.org/docs/master/data.html#torch.utils.data.Dataset). It needs a `__len__` method defined, which returns the size of the dataset, and a `__getitem__` method which returns the `i`th set of the eight inputs to the model.

The `Dataset` will be used by a PyTorch [`DataLoader`](https://pytorch.org/docs/master/data.html#torch.utils.data.DataLoader) in `train.py` to create and feed batches of data to the model for training or validation.

### Language Models

### Highway Networks

### Sequence Labeling Model

### Conditional Random Field (CRF)

# Training

See `train.py`.

### Viterbi Loss

### Viterbi Decoding

### Remarks

# Inference

### Some more examples

# FAQs

__How do we decide if we need <start> and <end> tokens for any model that uses sequences as inputs or targets?__

__Can we have the CRF output `current_word -> next_word` scores instead of `previous_word -> current_word` scores?

__Why are we using different vocabularies for the the inputs to the sequence tagger and language models' outputs?__

__Is it a good idea to fine-tune the pre-trained word embeddings we use in this model?__

__How do we use dynamic graphs in PyTorch to compute over only the true lengths of sequences?__