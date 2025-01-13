---
title: "Tokenization repair using Transformers"
date: 2021-05-28T11:44:13+01:00
author: "Sebastian Walter"
authorAvatar: "img/ada.jpg"
tags: ["nlp", "deep learning", "transformer", "tokenization", "repair"]
categories: []
image: "img/project_tokenization_repair_transformers/title_image.svg"
draft: false
---

This project tackles the tokenization repair problem using the Transformer neural network architecture. We achieve results that match the performance of previous work on multiple tokenization repair benchmarks paired with usable runtimes in practice.

<!--more-->

# Content

- [Problem](#problem)
    - [Applications](#applications)
    - [Challenges](#challenges)
- [Data](#data)
    - [Datasets](#datasets)
    - [Preprocessing](#preprocessing)
- [Architecture](#architecture)
- [Approach](#approach)
    - [How to repair sequences](#seqrepair)
    - [Neural Machine Translation](#nmt)
    - [Encoder-only](#eo)
    - [Training](#training)
- [Results](#results)
    - [Evaluation metrics](#metrics)
    - [Benchmarks](#benchmarks)
    - [Runtimes](#runtimes)
- [Visualisation](#visualisation)
- [Conclusion](#conclusion)
- [Demo](#demo)
- [References](#references)
- [Additional](#additional)

# Problem {#problem}

Tokenization repair refers to the process of correcting all whitespacing errors in text.

For example, given the sentence

`"I like toplay foo tball."`,

a good tokenization repair model would correct it to

`"I like to play football."`

## Applications

Repairing the whitespaces in text can be a helpful preprocessing or postprocessing step in various situations. 

Almost all NLP systems for example require their input text to be split into smaller units, often words or subwords, socalled tokens. For splitting text into tokens it is common practice to use a fixed set of rules, e.g. regular expressions, that solely or heavily rely on the whitespace information in the text. Therefore, missing or wrongly placed whitespaces can affect the performance of these NLP systems.

Another example where tokenization repair can be helpful is text extraction from images via OCR or from PDF documents. Since in both cases the text extraction procedure is error-prone with regard to whitespacing errors tokenization repair can be used as a postprocessing step to improve quality of the extracted texts [2]. 

## Challenges {#challenges}

Despite being a seemingly simple problem at first, there are some things that make tokenization repair challenging:

**1. Spelling errors**

Text in the real world often comes with spelling errors or uncommon language (e.g. think of text from tweets or messaging apps). This makes the problem more difficult since the number of possible words that can appear in the text grows very very large. Spelling errors also introduce ambiguities which can make it difficult to predict the correct whitespacing: `"we re"` for example could be both `"were"` or `"we are"`.

Note that in this blog post we look at methods that work on text with spelling errors. However, we do not correct these spelling errors. [2] argue that spelling correction and tokenization repair can indeed be seen as different problems. Many existing spell checkers for example do not correct the tokenization before correcting spelling errors which means that they require their input with spelling errors to have proper whitespacing. Therefore, it is crucial to incorporate spelling errors into the tokenization repair problem, but it is not necessary to correct them at the same time.

**2. Text with no whitespace information at all**
        
You may encounter texts like  `"Iliketoplayfootball"` where you can not make use of already existing whitespace information. This makes the problem a lot harder since most of the times existing whitespaces are already at correct positions and help you in determining word boundaries. This problem also appears with languages like Chinese, where text in general is written without whitespaces, leaving the reader responsible to determine the word boundaries. The problem of predicting the word boundaries from text with no whitespaces is called **word segmentation** [5]. 

Word segmentation can be seen as a special case of tokenization repair. We have to account for this while developing our models. Later more on this topic.
           
**3. Inference speed**

Prior works on tokenization repair or word segmentation (e.g. [2], [4], [5]) are mostly relying on neural and n-gram language models to score candidate segmentations. The idea is to search through the space of all possible segmentations using beam search and in the end taking the most likely segmenation. Therefore however, they have to go through the text sequentially token by token which is slow in practice. For practical usage a tokenization repair approach that could segment the whole text at once or could leverage parallelization would be beneficial. 

## Data {#data}

To our knowledge there are no datasets for the tokenization repair problem publicly available. However, training data for tokenization repair can be obtained by randomly introducing whitespacing errors into correct text.

### Datasets {#datasets}

As basis for our experiments we use the following two corpora to obtain a large collection of English texts:

- **English Wikipedia**: Articles from Wikipedia can be assumed to be error free in general, so we use them as examples for correctly tokenized and spelled text. Wikipedia also covers a wide range of different topics, which is good to train a general purpose tokenization repair model. However, it is biased towards academic and scientific style of writing.

    Dumps of the English Wikipedia can be obtained from [wikimedia.org](https://dumps.wikimedia.org/). In particular, we used this [dump](https://dumps.wikimedia.org/enwiki/20201020/enwiki-20201020-pages-articles-multistream.xml.bz2https://dumps.wikimedia.org/enwiki/20201020/enwiki-20201020-pages-articles-multistream.xml.bz2) from 20. October 2020 and extracted the text from the XML-files using [wikiextractor](https://github.com/attardi/wikiextractor).
    
- **Bookcorpus**: The Bookcorpus dataset contains a total of 17.868 books. The books are scraped from [smashwords.com](https://smashwords.com), one of the largest websites offering free e-books for download. By including the Bookcorpus into our training dataset we make our training data more diverse in terms of text styles since the Bookcorpus e.g. contains a lot of novels and science fiction books. The idea is that this counters the bias in Wikipedia towards academic and scientific articles while simply increasing the number training samples at the same time. 

    The Bookcorpus was obtained from the Huggingface dataset hub. It can be downloaded using the [Hugginface datasets library](https://github.com/huggingface/datasets) where it is called `bookcorpusopen`.

### Preprocessing {#preprocessing}

To clean the data we split all the articles from Wikipedia and the books from the Bookcorpus into sentences using the [spaCy sentencizer](https://spacy.io/api/sentencizer). We then remove all leading, trailing and doubled whitespaces as well as special characters like line breaks. We also unify quotation marks and fix Unicode errors. We exclude sentences that are empty, do not contain alphabetic characters, still contain XML markup (due to errors during extraction with the [wikiextractor](https://github.com/attardi/wikiextractor)) or are longer than 512 characters.

By this preprocessing procedure we obtain cleaned data:

| Dataset | Number of sentences |
| ------- | ------------------- |
| Bookcorpus | ~ 90.000.000 |
| Wikipedia | ~ 115.000.000 |
| Total | ~ 225.000.000

**Corrupting sequences**
After preparing and cleaning the data, we still have to create sequences with whitespacing and spelling errors from it. We do this in the following way for each sequence:

1. Introduce random spelling errors:
    - With probability 0.8: 
        
        Introduce a random spelling error in 15% of all words (split by whitespace) ¹
    
    - Else: 
        
        Keep sequence unchanged

2. Introduce whitespace errors:
    - With probability 0.8: 
        
        Introduce random whitespace errors into the sequence ²
    
    - Else: 
    
        Remove all whitespaces from the sequence

<details>
<summary>Details: Corruption function pseudocode</summary>

```python
def corrupt(sequence: str) -> Tuple[str, str]: 
    r = uniform(0, 1)
    if r >= 0.2:
        words = sequence.split(" ")
        edited_words = randomly_edit_words(words, prob=0.15)
        sequence = " ".join(edited_words)
        
    r = uniform(0, 1)
    if r >= 0.2:
        corrupted_sequence = randomly_insert_or_delete_whitespaces(sequence, insert_prob=0.1, delete_prob=0.2)
    else:
        corrupted_sequence = remove_all_whitespaces(sequence)
    
    return sequence, corrupted_sequence
```
</details>
<br>

Notice that the corrupted sequence we get after step 2 will be the input to our tokenization repair algorithm and the sequence after step 1 will be the corresponding groundtruth without whitespace errors. Also note that the groundtruth sequence indeed can contain spelling errors, since we only want to correct the whitespace errors as discussed in the [Challenges](#challenges) section earlier.

The following overview shows what kind of samples the training dataset will contain after applying the corruption function from above:
``` yaml
Input: 
This is a test sentence.

Possible corruptions:
Th isisa tes tsent ence.  # no spelling errors + whitespace errors => 16% of all sequences 
Thisisatestsentence.      # no spelling errors + no whitespaces    =>  4% of all sequences
Ti hsisa yes tsenti ence. # spelling errors    + whitespace errors => 64% of all sequences
Tihsisayestsentience.     # spelling errors    + no whitespaces    => 16% of all sequences
```

¹ *A random spelling error can be one of: Inserting a random character at a random position, deleting a random character, swapping two neighboring characters or replacing a character with a random character. From these four operations one is sampled uniformly and then applied to the word.*

² *A random whitespace error means either the deletion of an existing whitespace or the insertion of a whitespace before a non-whitespace character. For each non-whitespace character in the sequence we insert a whitespace before it with 0.1 probability and for each whitespace in the sequence we delete it with 0.2 probability.*

## Architecture {#architecture}

To tackle the tokenization repair problem we will adapt the popular Transformer Seq2Seq-architecture from [1]. At the core of the Transformer is the attention mechanism which allows an element in a sequence to get information about all other elements in the same or another sequence without the need for recurrent connections. This is helpful for problems where global information about the whole sequence is needed for prediction. It also helps with runtime performance because the attention mechanism can be computed for all sequence elements in parallel. Similar to other Seq2Seq-architectures that use recurrent neural networks, the Transformer consists of an encoder and a decoder. The encoder projects all elements of the input sequence into hidden representations in parallel. The decoder then uses these hidden representations and its own previous predictions to autoregressively predict the output sequence element by element (during training this can also be done in parallel). For more detailed and technical information about the Transformer see [1].

## Approach {#approach}

Based on the Transformer architecture introduced before we will model the tokenization repair problem in the following two ways:
1. Neural Machine Translation (NMT)
2. Encoder-only (EO)

Both approaches work by respecting existing whitespacing information in text. While tokenization repair can also be treated as a word segmentation problem where all whitespaces in a text are deleted up front, we found that keeping existing whitespaces greatly improves performance. Most likely this is because they contain a lot of information about the correct whitespacing. We rather wanted to solve the word segmentation problem as a subproblem of tokenization repair, which is also why about 20% of our training data contains no whitespacing (see [Preprocessing](#preprocessing)).

We will now have a closer look at the overall tokenization repair procedure and both individual approaches.

### How to repair sequences {#seqrepair}

With both the EO and NMT approaches we predict repair tokens to repair the input sequence. Note that we do not try to predict repaired text without tokenization errors directly by outputting characters, sub-words or words because that would leave the possibility open to change other things about the input text than its whitespacing.

The usage of repair tokens requires us to tokenize input text on a character level. Any other tokenization scheme would not allow for an easy way to predict insertions of whitespaces at any position in the text. However, this restriction is not a problem for tokenization repair in the way we formulated it, since tokenizing on character level also achieves two other important things (see [Challenges](#challenges)):
- character level tokenization fits naturally to text with spelling errors because all potentially misspelled words can be easily captured and represented without increasing the vocabulary size
- the low vocabulary size also reduces the number of parameters in the models which benefits inference time

The number of repair tokens is expected to be same as the length of the input text because for each character in the input text we predict exactly one repair token. We use three different repair tokens `0`, `1` and `2` where 

- `0` means that we keep the character at the same position in the input sequence unchanged,
- `1` means that we insert a whitespace before the character at the same position in the input sequence and
- `2` means that we delete the whitespace at the same position in the input sequence.

The following Python pseudocode shows the simple linear-time algorithm we use to repair sequences. Note that the `repair_tokens` argument is a list of `0`'s, `1`'s and `2`'s with the same length as the input sequence.

```python
def tokenization_repair(sequence: str, repair_tokens: List[int]) -> str:
    # code that checks whether inputs are valid is omitted for simplicity
    
    sequence_ptr = 0
    token_ptr = 0
    
    repaired_sequence = ""
    
    while sequence_ptr < len(sequence):
        char = sequence[sequence_ptr]
        prev_char = sequence[sequence_ptr - 1] if sequence_ptr > 0 else ""
        repair_token = repair_tokens[token_ptr]
        
        if repair_token == 1 and char != " " and prev_char != " ":
            # if we should insert a whitespace, make sure we only do it between two non-whitespace characters
            repaired_sequence += " " + char
            
        elif repair_token == 2 and char == " ":
            # if we should delete a whitespace, make sure we are at a whitespace and then just skip
            pass
        
        else:
            # add current character to repaired sequence in all other cases
            repaired_sequence += char
            
        sequence_ptr += 1
        token_ptr += 1
    
    return repaired_sequence
```

Let \\(n\\) be the length of the input sequence that should be repaired. It is easy to see that this algorithm is \\(\mathcal{O}(n)\\) since we look at each character in the input sequence exactly once. Its cost is negligible compared to the forward passes of the neural networks.

### NMT {#nmt}

The NMT approach leaves the original Transformer architecture unchanged. Inspired by Neural Machine Translation, where we translate a text from a source to a target language, we translate a text with tokenization errors into a repair sequence that will be later be used to correct the input text (see [How to repair sequences](#seqrepair)).

### EO {#eo}

The EO approach is motivated by the following idea: for every possible input text with tokenization errors there is exactly one sequence of repair tokens that repairs the input text correctly (ignoring the fact that e.g. predicting the deletion repair token `2` on a non-whitespace character behaves the same as the keep repair token `0`, since the model quickly learns to not do this). Based on this observation we assume that the prediction of a repair token for a certain character in the input text is independent of the prediction for all other characters since the correct repair token sequence is unique. In other words, to predict the repair token for some character in the input text the model does not need to be conditioned on the repair tokens that were predicted for previous characters because regardless of all previous predictions the correct prediction for the current character will not be influenced. Therefore, an autoregressive decoding procedure like it is used in the NMT approach might not be necessary for the tokenization repair problem. For the EO approach we drop the decoder entirely, which allows us to predict all repair tokens in parallel using just the encoder in a character level classification fashion.

### Training {#training}

For each approach we train three models of different sizes with respect to the number of parameters. The models are shown in the table below, where models prefixed with `EO` have an encoder-only architecture and models prefixed with `NMT` are based on the neural machine translation architecture. The number of layers for NMT models stand for the number of encoder and decoder layers respectively. All models suffixed with `wiki` are trained on the Wikipedia dataset only. We also trained one model on Wikipedia and Bookcorpus to study the effect of using more and more diverse training data.

| Model    | Layers | \# Parameters |
|:--------:|:------:|:-------------:|
| EO-small-wiki | 2      | 6,359,043    |
| EO-base-wiki  | 6      | 18,968,579    |
| EO-large-wiki | 12     | 37,882,883    |
| NMT-small-wiki | 1 + 1 | 7,414,793    |
| NMT-base-wiki | 3 + 3  | 22,127,625   |
| NMT-large-wiki | 6 + 6 | 44,196,873    |
| EO-base-wiki-bookcorpus | 6 | 18,968,579 |

**Training details**

The most important training information and hyperparameters are shown in the table below:

| Component              | Info                                                         |
|:----------------------:|:------------------------------------------------------------:|
| Optimizer              | AdamW [3]                                                 |
| Learning rate          | 0.0001 (and 0.01 weight decay)                              |
| Learning rate schedule | Cosine decay with linear warmup phase of 16000 steps         |
| Epochs                 | 4 (but we stop training after 48 hours max) ¹               |
| Batching               | [Bucket sampling](#bucket_sampling) |
| Validation data        | 20.000 randomly sampled sequences from the training data     |
| GPUs                   | 4 or 8 Nvidia Tesla V100 ²                           |

¹ *For the EO-base-wiki-bookcorpus model we only trained for 1 epoch because we used more training data.*

² *The GPUs were accessed through the bwUniCluster 2.0. We acknowledge support by the state of Baden-Württemberg through bwHPC.*

###### Bucket sampling {#bucket_sampling}

We do not use a fixed batch size, but set a maximum number of how many tokens can be in one batch. We then create buckets of training samples that have around the same length (difference between length of samples in one bucket is <= 4). For each step during training we pull out sequences from a random bucket until we hit the max token limit. This procedure creates batches that require minimal padding and therefore speeds up training and allows us   to use a large maximum sequence length of 512 throughout the whole training.

## Results {#results}

### Evaluation metrics {#metrics}

Let's look at the evaluation metrics we will use to assess the performance of our models. All of the evaluation metrics below work on pairs of strings \\((g_i, p_i)\\), where \\(p_i\\) is the prediction of our model and \\(g_i\\) is the groundtruth for the input sequence in the evaluation set at position \\(i\\). Let \\(N\\) be the number of sequences in the evaluation set, meaning \\(i \in \\{1,2,3,..., N\\}\\).

**F1, Precision and Recall**

Given a groundtruth sequence \\(g\\) and the predicted sequence \\(p\\), split both of them into tokens by whitespace to get two lists of tokens \\(g'\\) and \\(p'\\). 

Then let the number of true positives \\(\mathit{TP}\\) be the number of common tokens in \\(g'\\) and \\(p'\\). 
Let the number of false positives be \\(FP = |p'| - TP\\). 
Let the number of false negatives be \\(FN = |g'| - TP\\).

Then calculate Precision \\(\mathit{P}\\), Recall \\(\mathit{R}\\) and the F1-score \\(\mathit{F1}\\) as follows:

$$\mathit{P} = \frac{\mathit{TP}}{\mathit{TP} + \mathit{FP}} = \frac{\mathit{TP}}{|p'|}$$

$$\mathit{R} = \frac{\mathit{TP}}{\mathit{TP} + \mathit{FN}} = \frac{\mathit{TP}}{|g'|}$$

$$\mathit{F1} = \frac{2 \cdot \mathit{P} \cdot \mathit{R}}{\mathit{P} + \mathit{R}}$$

    Example:

    g = "This is a test sentence."
    p = "This isa test sentence."

    g' = ["This", "is", "a", "test", "sentence."]
    p' = ["This", "isa", "test", "sentence."]

    TP = |{"This", "test", "sentence."}| = 3
    FP = |p'| - TP = 4 - 3 = 1
    FN = |g'| - TP = 5 - 3 = 2

    P    = TP / (TP + FP) = 3 / 4 = 0.75
    R    = TP / (TP + FN) = 3 / 5 = 0.6
    F1   = (2 * P * R) / (P + R) = 0.9 / 1.35 = 0.666

**Sequence accuracy**

Sequence accuracy calculates the percentage of completely correct predicted sequences out of all predicted sequences.

Formally, sequence accuracy \\(\mathit{SA}\\) can be expressed as follows:

$$\mathit{SA} = \frac{1}{N} \sum\limits_{i=1}^{N} equal(g_i, p_i)$$ with $$equal(g, p) = \begin{cases} 1 & \text{ if } g = p \\\\ 0 & \text{ else }\end{cases}$$

    Example:

    g_1 = "This is a test sentence."
    p_1 = "This isa test sentence."

    g_2 = "I like to plai football."
    p_2 = "I like to plai football."

    g_3 = "The man wars a hat."
    p_3 = "The man wars a hat."

    g_4 = "Transformers are awwesome!"
    p_4 = "Transformers are awwesome!"

    seqacc = (1 / 4) * (0 + 1 + 1 + 1)  = 0.75

**MNED and MED**

Let \\(\mathit{ed}(g, p)\\) be a function that returns the edit distance between two strings \\(g\\) and \\(p\\). 

The edit distance calculates the number of character operations that are required to turn \\(g\\) into \\(p\\). A character operation can be inserting a character, deleting a character and replacing a character with another character.

We then calculate the mean edit distance \\(\mathit{MED}\\) and the mean normalized edit distance \\(\mathit{MNED}\\) between the groundtruths and predictions as follows:

$$\mathit{MED} = \frac{1}{N} \sum\limits_{i=1}^{N} \mathit{ed}(g_i, p_i)$$

$$\mathit{MNED} = \frac{1}{N} \sum\limits_{i=1}^{N} \mathit{ned}(g_i, p_i)$$ with $$\mathit{ned}(g, p) = \frac{\mathit{ed}(g, p)}{\max\\{ |g|, |p| \\}}$$

    Example:

    g_1 = "This is a test sentence."
    p_1 = "This isa test sentence."

    g_2 = "Transformers are awwesome!"
    p_2 = "Transformers are awwesome!"

    ed(g_1, p_1) = 1
    ed(g_2, p_2) = 0

    ned(g_1, p_1) = 1 / max(24, 23) = 1 / 24
    ned(g_2, p_2) = 0 / max(26, 26) = 0

    MED  = (1 / 2) * (1 + 0)        = 0.5
    MNED = (1 / 2) * ((1 / 24) + 0) = 0.0208

### Benchmarks {#benchmarks}

The models are evaluated on the tokenization repair benchmarks from [2], which can be retrieved from the [AD Freiburg tokenization repair repository](https://github.com/ad-freiburg/tokenization-repair), and on the Doval benchmark for word segmentation [4]. The following table shows the sequence accuracy achieved by our models on those benchmarks as well as the best model from [2] on each benchmark as reference. For detailed result tables including all metrics see the [Additional](#additional) section. Since the benchmarks are based on sequences from Wikipedia, we excluded all articles that are used in the benchmarks from our training data.

| *Metric: Sequence accuracy*<br>Model                    |   Doval | Wiki<br>(10% whitespacing<br> errors) |Wiki<br>(no whitespaces) | Wiki<br>(10% whitespacing <br>errors + typos) | Wiki<br>(no whitespaces <br>+ typos) |
|:-------------------------|--------------------:|------------:|------:|---------:|------------:|
| Best from [2]            | 0.964 | 0.9745 | 0.9447 | 0.9491 | 0.8341 |
| eo_large_wiki            | 0.952 | 0.9760 | 0.9492 | 0.9553 | 0.8648 |
| nmt_large_wiki           | 0.951 | 0.9749 | 0.9451 | 0.9530 | 0.8585 |
| eo_base_wiki             | 0.938 | 0.9703 | 0.9356 | 0.9387 | 0.8259 |
| nmt_base_wiki            | 0.937 | 0.9728 | 0.9414 | 0.9478 | 0.8464 |
| eo_base_wiki_bookcorpus  | 0.889 | 0.9615 | 0.9215 | 0.9233 | 0.8027 |
| eo_small_wiki            | 0.869 | 0.9482 | 0.8820 | 0.8834 | 0.7195 |
| nmt_small_wiki           | 0.854 | 0.9494 | 0.8851 | 0.8957 | 0.7510 |

The results show that the large variants of both the EO and NMT approach perform equally well or better than the best models from [2] on all tokenization repair benchmarks. On the Doval benchmark for word segmentation, our best model achieves a 0.012 lower sequence accuracy and a 0.0015 lower F1 score.

There is a noticable gap in performance between the large, base and small variants of the models. While for the large variants the EO approach always performs slightly better than the NMT approach, it is the opposite for the base and small variants.

As expected, the benchmarks with typos are harder than those without. The model performances in terms of sequence accuracy drop by about 2-4 percentage points for the benchmark with 10% whitespacing errors and typos and by about 10 percentage points or more for the benchmark with no whitespacing and typos. 

Interestingly, the benchmark with no whitespacing seems to be harder than the benchmark with 10% whitespacing errors and typos. This underlines the importance of using the existing whitespace information for tokenization repair.

The base model trained on Wikipedia and Bookcorpus performs worse than the base model trained only on Wikipedia. We think this is likely because all benchmarks are based on Wikipedia articles which have a different style of writing and topics than the books from the Bookcorpus. Comparing them on other benchmarks than Wikipedia is left for future work.

### Runtimes {#runtimes}

Inference runtimes are important for practical use of neural networks, so we need to take that into consideration next to the raw model performance. For detailed runtime tables see the [Additional](#runtimes) section.

The NMT models are expectedly slower than the EO models, since they predict the output repair sequence autoregressively rather than in parallel. Our fastest model `EO-small-wiki` can correct around 522 sequences per second when running inference on one sequence at a time. We can improve the runtime by additionally batching multiple sequences (preferably of similar length) together. With a batch of size 16 containing sequences of similar length `EO-small-wiki` is able to correct around 1967 sequences per second. In general the NMT models are by a factor of 50-100 slower than their corresponding EO counterparts.

## Visualisation {#visualisation}

We will visualise the `EO-base-wiki` model here to gain some insights into how the models work. All of the following visualisations are generated using the [demo](#demo). If you want to interactively visualise the other models too please have a look at it.

First we visualise the input embeddings for each character in the vocabulary and cluster them into four clusters with a standard K-Means algorithm. The result is not surprising but still interesting: the blue cluster contains all digits from 0 to 9, the yellow cluster mostly contains rarely used special characters, the green cluster contains all uppercase letters and the red cluster contains all lowercase letters and special tokens like `<bos>`, `<eos>` or `<unk>`. The model seems to learn sensible character representations.

![Embeddings](/../../img/project_tokenization_repair_transformers/embedding.svg)
*Figure 1: Encoder embeddings*

Going one step further, let's look at the token hidden representations coming out of each of the six layers of the `EO-base-wiki` model during a forward pass. We again color the representations, but this time not by clustering them, we instead define a color for each possible repair token `0`, `1` and `2`. We then assign the hidden representation of a character the color of its associated target repair token. As an example input sentence we will use `T his i s a sente nceI w an t to corr ec t.`. 

After applying the first layer the hidden representations are still clustered by character, most likely because they are still dominated by the input embeddings. Going from layer two to layer five the hidden representations are getting more and more aligned depending on the position in the sentence. This is not visible in figure 2, but in the demo you can verify this by simply hovering over the individual colored dots and checking the additional information that pops up. Finally, the output representations of the last layer show a clear separation by target repair token. This is what we would expect because this way the final linear projection layer of the `EO-base-wiki` model has an easy job to discriminate the three output classes.

![Hidden representations](/../../img/project_tokenization_repair_transformers/hidden_rep.svg)
*Figure 2: Encoder hidden representations*

We can also see some interesting patterns when looking at the attention maps generated by the encoder layers. Attention maps are a method to visualise from which other tokens in the sequence a token gets information to update its own hidden representation. The following is the attention map of layer 3 in the `EO-base-wiki` model. We use the same sentence from above as input. We can see that the model already in the middle layers pays almost no attention to whitespaces that should be removed, while whitespaces that should be kept get more attention.

![Attention map](/../../img/project_tokenization_repair_transformers/attention_middle_layer.svg)
*Figure 3: Attention map middle layer*

Some other interesting patterns can be observed in the attention maps of the last layer. For Figure 4 we removed all whitespaces from our original input sentence leaving it as `ThisisasentenceIwanttocorrect.` because this makes the pattern come out more clearly. \
It seems as if the model in the last layer pays attention especially to the beginning and ending of words, indicated by the red vertical bars in the attention map. This intuitively makes sense, because predicting the correct insertions and deletions of whitespaces is easy if one knows the exact word boundaries.

![Attention map](/../../img/project_tokenization_repair_transformers/attention_no_spaces.svg)
*Figure 4: Attention map last layer (no whitespaces)*

We now change our input to `T h i s i s a s e n t e n c e I w a n t t o c o r r e c t .`. In the attention map of the last layer we can still observe the tendency of the model to pay attention to word boundaries (vertical bars). Additionally we now can observe some kind of a checkerboard pattern along the main diagonal. This is because the whitespaces pay a lot of attention to their surrounding characters while the characters also pay some attention to their surrounding whitespaces.

![Attention map](/../../img/project_tokenization_repair_transformers/attention_all_spaces.svg)
*Figure 5: Attention map last layer (all whitespaces)*

For the NMT models similar patterns can be observed in the hidden representations as well as in the attention maps. Feel free to explore the visualisations more in depth in the [demo](#demo).

## Conclusion {#conclusion}

In this project we looked at two approaches to the tokenization repair problem for text with spelling errors that use the Transformer neural network architecture: NMT and EO. Both approaches achieve results that are on par with existing methods on multiple tokenization repair and word segmentation benchmarks. We demonstrated that by using only the encoder of a Transformer inference speed can be increased by a factor of 50-100 with little to no loss in performance, resulting in usable models in practice. 

## Demo {#demo}

This project comes with a demo in form of a [Streamlit](https://streamlit.io) application run inside a [docker](https://docker.com) container. The demo lets you explore all models described in this blog post. It includes functionality to interactively test and visualize the models, run and evaluate the models on all benchmarks mentioned in this blog post (you can also upload your own custom benchmarks) and more.

## References

1. Attention Is All You Need - Ashish Vaswani et al. (2017)
2. Tokenization Repair in the Presence of Spelling Errors - Hannah Bast, Matthias Hertel, Mostafa M. Mohamed (2020)
3. Decoupled Weight Decay Regularization - Ilya Loshchilov, Frank Hutter (2019)
4. Comparing Neural- and N-Gram-Based Language Models for Word Segmentation - Yerai Doval, Carlos Gómez-Rodríguez (2019)
5. Natural Language Corpus Data - Peter Norvig (2009)

## Additional {#additional}

### Full benchmarks

In the following you will find the complete benchmark tables. The models are sorted descending by sequence accuracy. 
All models not starting with either `eo` or `nmt` are from [2] and shown here for comparison.

#### Doval [4]

| Model                    |   Sequence accuracy |        MNED |   MED |       F1 |   Precision |   Recall |
|:-------------------------|--------------------:|------------:|------:|---------:|------------:|---------:|
| bidirectional_wikipedia  |               0.964 | 0.00053 | 0.05  | 0.9963 |    0.9971 | 0.9955 |
| unidirectional_wikipedia |               0.954 | 0.00069 | 0.059 | 0.9953 |    0.9954 | 0.9952 |
| eo_large_wiki            |               0.952 | 0.00061 | 0.072 | 0.9947 |    0.9953 | 0.9940 |
| nmt_large_wiki           |               0.951 | 0.00067 | 0.081 | 0.9944 |    0.9953 | 0.9935 |
| eo_base_wiki             |               0.938 | 0.00080 | 0.086 | 0.9934 |    0.9942 | 0.9926 |
| nmt_base_wiki            |               0.937 | 0.00089 | 0.1   | 0.9929 |    0.9938 | 0.9921 |
| doval                    |               0.922 | 0.00105 | 0.117 | 0.9909 |    0.9896 | 0.9923 |
| eo_base_wiki_bookcorpus  |               0.889 | 0.00182 | 0.142 | 0.9886 |    0.9905 | 0.9867 |
| eo_small_wiki            |               0.869 | 0.00194 | 0.186 | 0.9859 |    0.9867 | 0.9850 |
| nmt_small_wiki           |               0.854 | 0.00271 | 0.242 | 0.9828 |    0.9856 | 0.9800 |

#### Wiki (10% whitespacing errors) [2]

| Model                         |   Sequence accuracy |        MNED |       MED |       F1 |   Precision |   Recall |
|:------------------------------|--------------------:|------------:|----------:|---------:|------------:|---------:|
| eo_large_wiki                 |            0.9760 | 0.00045 | 0.028 | 0.9973 |    0.9972 | 0.9973 |
| nmt_large_wiki                |            0.9749 | 0.00047 | 0.030 | 0.9971 |    0.9971 | 0.9972 |
| bidirectional_wikipedia       |            0.9745 | 0.00043 | 0.030 | 0.9971 |    0.9967 | 0.9975 |
| nmt_base_wiki                 |            0.9728 | 0.00049 | 0.032 | 0.9969 |    0.9969 | 0.9969 |
| eo_base_wiki                  |            0.9703 | 0.00058 | 0.034 | 0.9967 |    0.9966 | 0.9968 |
| bidirectional_wikipedia_typos |            0.9695 | 0.00048 | 0.036 | 0.9965 |    0.9963 | 0.9968 |
| bidirectional_the_one         |            0.9684 | 0.00050 | 0.035 | 0.9965 |    0.9965 | 0.9965 |
| unidirectional_wikipedia      |            0.9656 | 0.00056 | 0.039 | 0.9962 |    0.9959 | 0.9965 |
| bidirectional_combo_errors    |            0.9640 | 0.00059 | 0.042 | 0.9959 |    0.9956 | 0.9962 |
| eo_base_wiki_bookcorpus       |            0.9615 | 0.00067 | 0.045 | 0.9957 |    0.9956 | 0.9957 |
| nmt_small_wiki                |            0.9494 | 0.00091 | 0.063 | 0.9941 |    0.9944 | 0.9938 |
| eo_small_wiki                 |            0.9482 | 0.00087 | 0.062 | 0.9941 |    0.9937 | 0.9944 |
| dynamic_programming           |            0.8617 | 0.00358 | 0.239 | 0.9807 |    0.9763 | 0.9851 |
| greedy                        |            0.7670 | 0.00517 | 0.433 | 0.9662 |    0.9626 | 0.9698 |
| wordsegment                   |            0.4113 | 0.02751 | 1.969 | 0.8735 |    0.8831 | 0.8641 |

#### Wiki (no whitespaces) [2]

| Model                         |   Sequence accuracy |       MNED |        MED |       F1 |   Precision |    Recall |
|:------------------------------|--------------------:|-----------:|-----------:|---------:|------------:|----------:|
| eo_large_wiki                 |            0.9492 | 0.00105 |  0.070 | 0.9938 |    0.9940 | 0.9936  |
| nmt_large_wiki                |            0.9451 | 0.00118 |  0.080 | 0.9932 |    0.9934 | 0.9930  |
| bidirectional_wikipedia       |            0.9447 | 0.00102 |  0.074 | 0.9933 |    0.9931 | 0.9935  |
| nmt_base_wiki                 |            0.9414 | 0.00127 |  0.086 | 0.9927 |    0.9931 | 0.9923  |
| eo_base_wiki                  |            0.9356 | 0.00131 |  0.092 | 0.9920 |    0.9923 | 0.9917  |
| bidirectional_wikipedia_typos |            0.9353 | 0.00131 |  0.089 | 0.9922 |    0.9922 | 0.9922  |
| unidirectional_wikipedia      |            0.9223 | 0.00161 |  0.111 | 0.9903 |    0.9893 | 0.9913  |
| eo_base_wiki_bookcorpus       |            0.9215 | 0.00152 |  0.111 | 0.9902 |    0.9906 | 0.9899  |
| bidirectional_combo_errors    |            0.9187 | 0.00164 |  0.115 | 0.9899 |    0.9901 | 0.9898  |
| nmt_small_wiki                |            0.8851 | 0.00229 |  0.178 | 0.9853 |    0.9864 | 0.9843  |
| eo_small_wiki                 |            0.8820 | 0.00224 |  0.174 | 0.9849 |    0.9845 | 0.9854  |
| dynamic_programming           |            0.8616 | 0.00358 |  0.239 | 0.9807 |    0.9763 | 0.9851  |
| wordsegment                   |            0.4113 | 0.02751 |  1.969 | 0.8735 |    0.8831 | 0.8641  |
| greedy                        |            0.1296 | 0.11875 | 12.591 | 0.1085 |    0.3612 | 0.06383 |

#### Wiki (10% whitespacing errors + typos) [2]

| Model                         |   Sequence accuracy |        MNED |       MED |       F1 |   Precision |   Recall |
|:------------------------------|--------------------:|------------:|----------:|---------:|------------:|---------:|
| eo_large_wiki                 |            0.9553 | 0.00075 | 0.058 | 0.9949 |    0.9949 | 0.9949 |
| nmt_large_wiki                |            0.9530 | 0.00084 | 0.064 | 0.9945 |    0.9945 | 0.9945 |
| bidirectional_the_one         |            0.9491 | 0.00076 | 0.061 | 0.9943 |    0.9944 | 0.9942 |
| nmt_base_wiki                 |            0.9478 | 0.00096 | 0.073 | 0.9938 |    0.9938 | 0.9938 |
| bidirectional_combo_errors    |            0.9463 | 0.00084 | 0.068 | 0.9938 |    0.9935 | 0.9942 |
| bidirectional_wikipedia_typos |            0.9443 | 0.00092 | 0.075 | 0.9935 |    0.9931 | 0.9939 |
| eo_base_wiki                  |            0.9387 | 0.00103 | 0.080 | 0.9929 |    0.9928 | 0.9931 |
| eo_base_wiki_bookcorpus       |            0.9233 | 0.00124 | 0.100 | 0.9911 |    0.9910 | 0.9912 |
| nmt_small_wiki                |            0.8957 | 0.00175 | 0.150 | 0.9873 |    0.9878 | 0.9867 |
| eo_small_wiki                 |            0.8834 | 0.00175 | 0.151 | 0.9864 |    0.9859 | 0.9869 |
| bidirectional_wikipedia       |            0.8329 | 0.00230 | 0.219 | 0.9802 |    0.9793 | 0.9810 |
| unidirectional_wikipedia      |            0.8024 | 0.00265 | 0.258 | 0.9761 |    0.9746 | 0.9775 |
| greedy                        |            0.4292 | 0.01299 | 1.190 | 0.8968 |    0.8779 | 0.9166 |
| dynamic_programming           |            0.4089 | 0.01699 | 1.550 | 0.8838 |    0.8528 | 0.9171 |
| wordsegment                   |            0.2046 | 0.04159 | 3.402 | 0.7819 |    0.7855 | 0.7784 |

#### Wiki (no whitespaces + typos) [2]

| Model                         |   Sequence accuracy |       MNED |       MED |        F1 |   Precision |    Recall |
|:------------------------------|--------------------:|-----------:|----------:|----------:|------------:|----------:|
| eo_large_wiki                 |            0.8648 | 0.00277 |  0.240 | 0.9817  |    0.9816 | 0.9818  |
| nmt_large_wiki                |            0.8585 | 0.00303 |  0.263 | 0.9805  |    0.9806 | 0.9803  |
| nmt_base_wiki                 |            0.8464 | 0.00340 |  0.288 | 0.9787  |    0.9790 | 0.9784  |
| bidirectional_wikipedia_typos |            0.8341 | 0.00351 |  0.298 | 0.9774  |    0.9772 | 0.9776  |
| bidirectional_combo_errors    |            0.8272 | 0.00374 |  0.311 | 0.9761  |    0.9763 | 0.9758  |
| eo_base_wiki                  |            0.8259 | 0.00345 |  0.299 | 0.9763  |    0.9759 | 0.9768  |
| eo_base_wiki_bookcorpus       |            0.8027 | 0.00384 |  0.340 | 0.9729  |    0.9724 | 0.9733  |
| nmt_small_wiki                |            0.7510 | 0.00529 |  0.481 | 0.9638  |    0.9653 | 0.9623  |
| eo_small_wiki                 |            0.7195 | 0.00530 |  0.481 | 0.9604  |    0.9589 | 0.9620  |
| bidirectional_wikipedia       |            0.6647 | 0.00654 |  0.628 | 0.9496  |    0.9495 | 0.9496  |
| unidirectional_wikipedia      |            0.5882 | 0.00931 |  0.887 | 0.9309  |    0.9258 | 0.9361  |
| dynamic_programming           |            0.4089 | 0.01699 |  1.550 | 0.8838  |    0.8528 | 0.9171  |
| wordsegment                   |            0.2046 | 0.04159 |  3.402 | 0.7819  |    0.7855 | 0.7784  |
| greedy                        |            0.1138 | 0.12018 | 12.656 | 0.09819 |    0.3295 | 0.05765 |

### Full runtimes

In the following you will find runtime measurements of all seven models in terms of total runtime, samples per second and seconds per kilobyte text. The runtime measurements were carried out over all of the five benchmarks above. For example, the total runtime tells you how long it took a model to complete all five benchmarks. The runtimes were measured on a Nvidia GeForce GTX 1080 Ti GPU.

#### Single batching

Single batching refers to running the models with a batch size of 1.

| Model                   |   Total runtime in seconds |   samples/s |      s/KB |
|:------------------------|---------------------------:|------------:|----------:|
| eo_small_wiki |                    78.47 |     522.29 | 0.022 |
| eo_base_wiki |                    168.20 |     243.67 | 0.048 |
| eo_large_wiki |                    308.20 |     132.98 | 0.089 |
| nmt_small_wiki |                    7737.16 |     5.29 | 2.234 |
| nmt_base_wiki |                    12827.8 |     3.19 | 3.704 |
| nmt_large_wiki |                    19998.8 |     2.04 | 5.775 |
| eo_base_wiki_bookcorpus |                    160.84 |      254.83 | 0.046 |

#### Smart batching with batch size 16

Smart batching means that we sort the sequences of each benchmark by their length before grouping them into batches. We thereby minimize the amount of padding required when running batched inference.

| Model                   |   Total runtime in seconds |   samples/s |      s/KB |
|:------------------------|---------------------------:|------------:|----------:|
| eo_small_wiki |                    20.83 |     1967.48 | 0.006 |
| eo_base_wiki |                    40.50 |     1012.02 | 0.011 |
| eo_large_wiki |                    69.10 |     593.11 | 0.019 |
| nmt_small_wiki |                     1221.4 |     33.55 | 0.352 |
| nmt_base_wiki |                    2222.93 |     18.43 | 0.641 |
| nmt_large_wiki |                    3867.06 |     10.59 | 1.116 |
| eo_base_wiki_bookcorpus |                     40.18 |     1019.98 | 0.011 |
