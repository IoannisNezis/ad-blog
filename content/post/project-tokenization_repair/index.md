---
title: "Tokenization Repair Using Character-based Neural Language Models"
date: 2019-02-25T15:15:43+01:00
author: "Matthias Hertel"
authorAvatar: "img/project_tokenization_repair/hertel.png"
tags: ["ml", "nlp", "rnn", "tokenization", "spelling"]
categories: ["project"]
image: "img/project_tokenization_repair/header.png"
draft: false
---
Common errors in written text are split words ("algo rithm") and run-on-words ("runsin").
I use Character-based Neural Language Models to fix such errors in order to enable further processing of misspelled texts.
<!--more-->

# Content

1. <a href="#introduction">Introduction</a>
1. <a href="#problem">Problem definition</a>
1. <a href="#datasets">Data sets</a>
1. <a href="#errormodel">Error model</a>
1. <a href="#metrics">Evaluation metrics</a>
1. <a href="#commercial_baselines">Commercial baselines</a>
1. <a href="#baseline">Dictionary-based baseline</a>
1. <a href="#methods">Character-based Neural Language Model Methods</a>
1. <a href="#language_model_evaluation">Language Model Evaluation</a>
1. <a href="#tokenization_repair_evaluation">Tokenization Repair Evaluation</a>
1. <a href="#spelling">Spelling Correction</a>
1. <a href="#summary">Summary</a>
1. <a href="#webapp">Web Application</a>

# <a id="introduction"></a> Introduction

The first step of almost every Natural Language Processing system is to split raw text into words. This task is called Tokenization.
Tokenization is usually done based on a language-specific set of handcrafted rules<a href="#footnote_spacy">¹</a>.
While state-of-the-art Tokenizers like the SpaCy tokenizer and Stanford CoreNLP Tokenizer work well on syntactically correct input,
they are not able to correctly recognize words in incorrect sequences.

Wrong spaces or missing spaces are frequent errors in written texts for several reasons, like Optical Character Recognition errors, typos or document formattings that spread single words over multiple lines.
<details>
<summary>DETAILS: examples demonstrating the SpaCy and CoreNLP Tokenizers' vulnerability to typos.</summary>

The following table shows examples for the vulnerability of the two previously mentioned tokenizers to wrong and missing spaces.

| __Input__ | __Spacy tokenization__ | __Stanford CoreNLP tokenization__ |
| --- | --- | --- |
| The algorithm runs in linear time. | \[The, algorithm, runs, in, linear, time, .\] | \[The, algorithm, runs, in, linear, time, .\] |
| The algo rithm runsin linear time. | \[The, algo, rithm, runsin, linear, time, .\] | \[The, algo, rithm, runsin, linear, time, .\] |
| This is a correct sequence. | \[This, is, a, correct, sequence, .\] | \[This, is, a, correct, sequence, .\] |
| Thisis anerro nous sequence. | \[Thisis, anerro, nous, sequence, .\] | \[Thisis, anerro, nous, sequence, .\] |
| Hello world. | \[Hello, world, .\] | \[Hello, world, .\] |
| Hel low orld. | \[Hel, low, orld.\] | \[Hel, low, orld.\] |

Both Tokenizers work well on the syntactically correct sequences (rows 1, 3, and 5), but introduce incorrect tokens in the other cases (rows 2, 4 and 6).
</details>

The following graph shows this project's final model in action. When fed with the sequence *Th eal go rithm runsin line art im e .*, it deletes spaces (red bars) and inserts spaces (green bars) to transform it into *The algorithm runs in linear time.*.

<img src="/../../img/project_tokenization_repair/tokenization_repair_correction.png"></img>

<a id="footnote_spacy"></a>¹ See, for example, the description of the [SpaCy tokenizer](https://spacy.io/usage/linguistic-features#section-tokenization).

# <a id="problem"></a> Problem definition

Given a potentially incorrect text sequence, the task is to transform it into a correct one, such that a Tokenizer can determine the correct tokens.

I only consider spaces as token delimiters, but one could use a bigger delimiter character set, for example also containing the hyphenation due to a line break (*the algo-\[newline\]rithm*).

In this project I focus on space insertions and space deletions as error types.
Space shifting (*thea lgorithm*) is not adressed directly, since a shifting operation can be modeled as a combination of an insertion operation and a deletion operation.
<details>
<summary>DETAILS: formal task definition.</summary>

More formal, given a sequence of characters
$$S = \[c_1, c_2, ..., c_n\]$$
and a set of delimiter characters \\(C\\),
the task is to find the set of positions
$$I \subseteq \\\{ i \hspace{5px} | \hspace{5px} 1 \leq i \leq n \hspace{5px} \land \hspace{5px} c_i \in C \\\}$$
where delitimers were inserted by mistake, and the set of positions
$$D \subseteq \\\{i \hspace{5px} | \hspace{5px} 0 \leq i \leq n\\\}$$
where delimiters were deleted by mistake.

The sets \\(I\\) and \\(D\\) can then be used to transform the given incorrect sequence into the correct sequence,
by inserting delimiters at the positions in \\(D\\) and deleting delimiters at the positions in \\(I\\).
</details>

In the end I analyse whether the methods developed for Tokenization Repair are also applicable to more general Text Correction, in particular to find and correct insertions and deletions of arbitrary characters.

# <a id="datasets"></a> Data sets

During the project I used the English part of the [Europarl German-English Parallel Corpus](http://www.statmt.org/europarl/) and a dump of the [English Wikipedia](https://dumps.wikimedia.org/) as data sets.
Some earlier evaluations were done on the Parallel Corpus, whereas Wikipedia was used for the final evaluation.

Each data set was split into a training set used to train models, a smaller development set used to tune the methods, and a test set that is only used for the final evaluation.

## Europarl Parallel Corpus

The German-English Parallel Corpus contains transcripts of the European Parliament's public meetings.
I only use the English part of the data set.
Each line is treated as a separate sequence that contains exactly one sentence.

## Wikipedia

The articles' texts were extracted from the Wikipedia dump using [Attardi's WikiExtractor script](https://github.com/attardi/wikiextractor).
The resulting files contain a full paragraph in each line, which I use as sequences without further splitting (since sentence splitting relies on Tokenization).
<details>
<summary>DETAILS: Wikipedia data quality (found artefacts).</summary>

I noticed that the extracted texts sometimes contain artefacts like user comments with timestamps, HTML code, web adresses, empty brackets, typos and foreign languages, but those artefacts are rare enough that the data could still be used to train a meaningful language model.

A few examples of artefacts in the extracted texts:

* Thank you for your time, -- () 19:14, 27 May 2010 (UTC)
* \[EDIT—removed a repetition\]
* HOW DO I ACESSS "People also searched for" feature for this page? 17:56, 11 October 2015 (UTC)
* !colspan=9| 2012 SWAC Men's Basketball Tournament
* () is located in Pakistan.
* \<onlyinclude\>\</onlyinclude\>
</details>
<details>
<summary>DETAILS: statistical comparison of the two data sets.</summary>

## Data set statistics

The following table summarizes the two data sets.
A simple tokenization, which splits sequences on spaces, was used to create the token statistics.
The term *coverage* refers to the number of (most frequent) unique tokens that make up a given percentage of all tokens in the corpus.

|     | Europarl EN | Wikipedia EN |
| --- | --- | --- |
| raw text size | 287 MB | 12.7 GB |
| sequences | 1,910,670 | 38,585,542 |
| tokens | 47,880,248 | 2,058,288,849 |
| unique tokens | 204,800 | 24,670,557 |
| 90% coverage | 5,233 tokens | 65,576 tokens |
| 99% coverage | 61,673 tokens | 6,678,921 tokens |

Wikipedia is much bigger and also much more diverse, which can be concluded from the higher number of tokens needed to get a 90% or 99% coverage.
The Parallel Corpus is rather homogeneous, with many words from politics appearing in the 50 most frequent tokens (*I, European, Mr, our, Commission, President* and *Member*).
</details>

# <a id="errormodel"></a> Error Model

In order to evaluate the performance of a Tokenization Repair method, pairs of misspelled sequences with their corresponding correct sequences are needed.
Since I am not aware of such a dataset for the task of Tokenization Repair, I created artificial examples using an error model that manipulates a token with probability \\(p\\).
<details>
<summary>DETAILS: probabilistic error model.</summary>

The error model has a parameter \\(p\\) that controls the probability for a token to be corrupted.
It takes a correct sequence from the development set or test set of one of the datasets as input, removes each delimiter with probability \\( \frac{p}{2} \\), and splits each token with probability \\( \frac{p}{2} \\) by inserting a space at a random position.
The resulting sequence is the *corrupted sequence* and the original sequence is the *correct sequence*.
The operations executed by the error model define the delimiter insertion positions \\(I\\) and delimiter deletion positions \\(D\\).
</details>

# <a id="metrics"></a> Evaluation Metrics

## F1 Score

Tokenization Repair can be seen as two separate binary classification tasks. At each position of a sequence, a prediction has to be made whether a delimiter was deleted, and (given the character at that position is a delimiter) whether a delimiter was inserted.
Since only a few characters get corrupted by the error model, analysing the accuracy of a method on those two classification tasks would not be a good performance metric - just predicting no corruption at every position would already give a high accuracy.
Instead, a method's *precision*, *recall* and *F1 score* on both subtasks shall be analysed.
<details>
<summary>DETAILS: definitions of true positives, false positives, false negatives, precision, recall and F1.</summary>

As defined above, \\(I\\) is the set of positions where a delimiter was erronously inserted
and \\(D\\) is the set of positions where a delimiter was erronously deleted.
Let \\(J\\) be the positions where the method predicts that a delimiter was inserted
and \\(T\\) the positions where it predicts that a delimiter was deleted.
Then consider the following definitions of true positives \\(\mathit{TP}\\), false positives \\(\mathit{FP}\\) and false negatives \\(\mathit{FN}\\), for the two subtasks of finding inserted spaces \\(I\\) and deleted spaces \\(D\\):
$$\mathit{TP_I} = | I \cap J |$$
$$\mathit{FP_I} = | J \setminus I |$$
$$\mathit{FN_I} = | I \setminus J |$$
$$\mathit{TP_D} = | D \cap T |$$
$$\mathit{FP_D} = | T \setminus D |$$
$$\mathit{FN_D} = | D \setminus T |$$

In other words, a true positive is a typo that is corrected by the method.
A false positive is an operation suggested by the method that inserts a new typo into the sequence.
A false negative is a typo that is not corrected by the method.

Using those definitions, *precision* and *recall* can be defined.
Precision \\(P\\) is the fraction of correct predictions among the method's predictions. Given the model suggests an operation, how likely is it to be correct?
Recall \\(R\\) is the fraction of typos that are corrected by the method. Given a sequence contains a typo, how likely does the method find it?
$$P_I = \frac{ \mathit{TP_I} }{ \mathit{TP_I} + \mathit{FP_I} }$$
$$R_I = \frac{ \mathit{TP_I} }{ \mathit{TP_I} + \mathit{FN_I} }$$
$$P_D = \frac{ \mathit{TP_D} }{ \mathit{TP_D} + \mathit{FP_D} }$$
$$R_D = \frac{ \mathit{TP_D} }{ \mathit{TP_D} + \mathit{FN_D} }$$

Clearly there is a tradeoff between precision and recall. Therefore the F1 score is used as a performance metric that combines both to a single value by taking the harmonic mean.
$$\mathit{F_{1,I}} = \frac{2 \cdot P_I \cdot R_I}{P_I + R_I}$$

$$\mathit{F_{1,D}} = \frac{2 \cdot P_D \cdot R_D}{P_D + R_D}$$

Since the Tokenization Repair methods must solve both subtasks together, it makes sense to define joint precision, recall and F1 measures.
$$P = \frac{ \mathit{TP_I} + \mathit{TP_D} }{ \mathit{TP_I} + \mathit{FP_I} + \mathit{TP_D} + \mathit{FP_D} }$$
$$R = \frac{ \mathit{TP_I} + \mathit{TP_D} }{ \mathit{TP_I} + \mathit{FN_I} + \mathit{TP_D} + \mathit{FN_D} }$$
$$F_1 =\frac{2 \cdot P \cdot R}{P + R}$$

Precision, recall and F1 score range from 0 to 1, where 1 is best.
</details>

## Edit Distance

Another way to evaluate the Tokenization Repair methods is to compare the edit distance to the ground truth sequence before and after correcting the input sequence.
<details>
<summary>DETAILS: Levenshtein edit distance.</summary>

The Levenshtein edit distance between sequences *A* and *B* is defined as the minimum number of edit operations needed to transform *A* into *B*, where an edit operation can either insert a character, delete a character or replace a character by another one.
</details>

For the mean edit distance \\(\mathit{E\_{before}}\\) between corrupted sequences and ground truth sequences, and the mean edit distance \\(\mathit{E\_{after}}\\) between predicted sequences and ground truth sequences, I define the *fraction of resolved edit operations* \\(F\\):
$$F = 1 - \frac{ \mathit{E\_{after}} }{ \mathit{E\_{before}} }$$
The fraction of resolved edit operations is negative when a method introduces more errors than it fixes, and between 0 and 1 otherwise, where 1 is best.

# <a id="commercial_baselines"></a> Commercial Baselines

## Google and LibreOffice

<img src="/../../img/project_tokenization_repair/google_tokenization_baseline.png" title="Usage of Google's spellchecker for Tokenization Repair."></img>

I compare my methods with the Google and LibreOffice spellcheckers.
To do so, I copy the corrupted sequence into a Google document or LibreOffice Writer document and apply all suggested changes that only include insertions or deletions of spaces.
The predicted edit operations are then computed using the backtrace of the edit distance matrix between the predicted and corrupted sequence.

## Aspell

The [GNU Aspell spellchecker](http://aspell.net/) flags incorrect words in a text and suggests corrections.
For Tokenization Repair, I apply corrections that include only space operations.
Unlike Google and LibreOffice, Aspell does not merge tokens.
To bypass this drawback, I check for each token flagged as a typo whether Aspell accepts the token merged with the preceeding or succeeding token, and if so merge it.

# Dictionary-based Baseline <a id="baseline"></a>

A straightforward idea to adress the Tokenization Repair problem is to use a dictionary of correctly spelled words to distinguish between words (tokens contained in the dictionary) and nonwords (tokens not contained in the dictionary).
Tokenization Repair then becomes the task to reorganize a sequence's non-delimiter characters into tokens such that the number of nonwords is minimal, while not applying an unrealistic number of edit operations.

## Dictionary Design and Pretokenization Scheme

Assuming that the training data is unlikely to contain the same misspellings multiple times,
a dictionary of correct words is created by storing the \\(k\\) most frequent tokens.

To avoid mixing words and punctuation marks into a single token, a rule-based pretokenization scheme is applied that identifies punctuation marks.
The frequencies of the resulting tokens in the training set are counted and the \\(k\\) most frequent tokens form the dictionary.
<details>
<summary>DETAILS: rule-based pretokenization scheme.</summary>

A simple pretokenization scheme would split a sequence on delimiters and treat all characters in between as a single token.
That precedure generates tokens that mix characters and punctuation marks (for example, "world", "world." and "world," would be treated as three distinct tokens).
Another idea is to split the sequence on all characters that are neither letters nor digits, but that would split compound words like "character-based".

Therefore I designed a rule-based tokenization scheme, that splits a sequence on blankspaces and all characters which are not in a-z, A-Z or 0-9, with the following exceptions:

* a comma or point between digits (e.g. *100,000.123*)
* an apostroph after a letter and before an *s* (e.g. *William's*)
* an apostroph after a letter and before a *t* (e.g. *wasn't*)
* an apostroph between *I* and *m* (*I'm*)
* a hyphen between letters (e.g. *character-based*)

At test time, the same tokenization scheme is applied to the input sequence, and the different parts between punctuation marks are treated as independent subproblems.
That is, for example, the sequence *I use charac ter-based languagemodels, because it's fun.* is split into two subproblems containing the tokens \[*I*, *use*, *charac*, *ter-based*, *languagemodels*\] and \[*because*, *it's*, *fun*\], separated by a comma and a point as punctuation marks.
</details>
<details>
<summary>DETAILS: naive dictionary-based Tokenization Repair approach.</summary>
## Naive approach

Given a pretokenized sequence like \[*I*, *use*, *charac*, *ter-based*, *languagemodels*\], the task is to reorganize the characters into a more likely token sequence.
For the baseline I assume that that the likeliness of a token sequence can be measured using the token frequencies.
Especially reducing the number of nonwords (tokens that do not appear in the dictionary) should increase the likeliness of a sequence.
For example, *charac* and *ter-based* are nonwords, but *character-based* is in the dictionary, so it is a good idea to merge the two tokens.
For the same reason, the nonword *languagemodels* should be split into the two words *language* and *models*.

A simple approach is to locate all \\(n\\) words that appear somewhere in the merged sequence *Iusecharacter-basedlanguagemodels* and generate \\(2^n\\) tokenization candidates from all subsets (ignoring subsets where words overlap).
The candidates are ranked by the number of nonwords in increasing order, and the best candidate is returned.

<details>
<summary>DETAILS: candidate generation and ranking example.</summary>

For example, the candidate for the words *I*, *use*, *act*, *mode* is \[*I*, *use*, *char*, *act*, *ter-basedlanguage*, *mode*, *ls*\].
The candidates can be rated using the number of nonwords, number of edit operations needed to generate it and frequencies of the tokens.
Indeed, the best candidate for the example sequence is \[*I*, *use*, *character-based*, *language*, *models*\], with zero nonwords and two edit operations (merging *character-based* and splitting *language models*).
</details>

A problem of this approach is the exponential increase of the number of candidates for longer sequences.
Using a dictionary with 1,000,000 words, 78 words are located in the example sequence from above, leading to \\(2^{78}\\) candidates, which is clearly intractable.
This means the naive approach can only be used on very short sequences (e.g. solving the tokenization problem for a token only based on its left and right neighbors), or using a very small dictionary, and motivates the use of a better alternative - the dynamic programming approach.
</details>

## Dynamic programming approach

The dynamic programming Tokenization Repair approach first locates all words in the character sequence ignoring all spaces, and then uses the words and present spaces to solve the Tokenization Repair task from left to right.

At position \\(i\\), it finds the best token split for the subsequence containing the first \\(i\\) characters, based on the already computed solutions for its prefixes and the words ending at position \\(i\\).
For each word \\(w\\) ending at \\(i\\), a candidate is generated, that consists of the best solution for the prefix until the beginning of \\(w\\) and the word \\(w\\).
Another candidate is the best solution until the last delimiter and the nonword ranging from the last delimiter to position \\(i\\).
The candidate with the least number of nonwords is selected for position \\(i\\), using the number of edit operations and the token frequencies from the least frequent to the most frequent token as tiebreakers.
<details>
<summary>DETAILS: dictionary-based dynamic programming approach example.</summary>

Example: the example sequence *I use charac ter-based languagemodels* gets merged into *Iusecharacter-basedlanguagemodels*, while storing the original delimiter positions. At the position of the *d*, the following candidates are generated for the subsequence *Iusecharacter-based*:

1. for the word *character-based*: best solution for *I use* + *character-based* = \[*I*, *use*, *character-based*\]
1. for the word *based*: best solution for *I use charac ter-* + *based* = \[*I*, *use*, *character-*, *based*\]
1. for the nonword *ter-based*: best solution for *I use charac* + *ter-based* = \[*I*, *use*, *char*, *ac*, *ter-based*\]

Among the candidates, \[*I*, *use*, *character-based*\] has the least nonwords (zero), and is picked as the best solution for position \\(i\\).
</details>

Locating all words in the merged sequence is done in *sequence length \* length of longest dictionary entry* time,
and each word is used exactly once to create a candidate, so the algo rithm runsin linear time.

## Postprocessing

Two postprocessing methods were introduced to improve the baseline.
One deals with spaces immediately before or after punctuation marks and is rule-based, and the other uses a decision rule, Support Vector Classifier or Random Forest model to resolve ambiguous cases based on the token frequencies.
<details>
<summary>DETAILS: punctuation mark postprocessing.</summary>

### Postprocessing part 1: punctuation marks

The baseline as described above does not deal with delimiters immediately before or after punctuation marks.
This is done in a postprocessing step based on the following rules, that were found to improve the results while having high precision:

* No delimiter follows a comma, point or closing bracket.
* There are no delimiters allowed between digits.
* Commas and semicolons have to be followed by a delimiter.
* Opening brackets have to be preceded by a delimiter.
</details>
<details>
<summary>DETAILS: ambiguity postprocessing.</summary>
### Postprocessing part 2: ambiguities

An analysis of the baselines' wrong predictions showed that it gets a lot of ambiguous cases wrong.
Such are valid words which can be split to two valid words, or two words which can be merged to a valid word - for example *carefully* / *care fully*, *into* / *in to* and *along* / *a long*.

I designed a postprocessing method that deals with the beforementioned ambiguities based on the frequencies of the merged token (*carefully*) and the two split tokens (*care* and *fully*).
Three versions were implemented:

1. A simple decision rule: use the split tokens if and only if both tokens are more frequent than the merged token.
1. A binary Support Vector Classifier (SVC) that takes the three token frequencies as input. It is trained on ambiguous cases from the training set.
1. Like the SVC version, but using a Random Forest instead of an SVC. That allows for more training data than the SVC.
</details>

# <a id="methods"></a> Character-based Neural Language Model Methods

A Character-based Neural Language Model estimates a probability distribution over a vocabulary of characters at each position in a sequence.
Those probabilities can be used for Tokenization Repair by classifying delimiters as wrongly inserted if the model predicts that the delimiter is unlikely, and classifying positions as wrong delimiter deletions if the model predicts that a delimiter is likely to occur there but is not present in the given sequence.

I distinguish between unidirectional language models, that only use the sequence in forward or backward direction to predict the next character, i.e. the forward model
$$p(c| \mathit{sequence} ) = p\_{\rightarrow}(c| \mathit{prefix} )$$
and the backward model
$$p(c| \mathit{sequence} ) = p\_{\leftarrow}(c| \mathit{suffix} )$$
and bidirectional language models, that use the whole sequence except for the target position:
$$p(c| \mathit{sequence} ) = p\_{\leftrightarrow}(c| \mathit{prefix}, \mathit{suffix})$$

## Character encoding

For the neural network models, characters are represented as one-hot-encoded vectors.
The character set consists of the 100 most frequent characters, and three extra symbols to encode the beginning and end of a sequence and out-of-set characters.

## Unidirectional models

The unidirectional models process a sequence either in forward or backward direction, predicting the next symbol at each position in the sequence.

The unidirectional models consist of a Long Short-Term Memory (LSTM) cell followed by a dense layer and a softmax output layer of dimension 103.

For visualising the information flow in the neural models, the following scheme is used:

| symbol | meaning |
| --- | --- |
| <img src="/../../img/project_tokenization_repair/legend_big_h.png" title="One-hot-vector (0 0 0 ... 0 1 0 ... 0 0 0) encoding the character 'H'."></img> | Yellow boxes represent one-hot-encoded input symbols. |
| <img src="/../../img/project_tokenization_repair/legend_small_h.png" title="hidden state"></img> | Yellow circles *h1* to *h6* represent hidden state vectors. |
| <img src="/../../img/project_tokenization_repair/legend_p.png" title="predicted probability distribution vector over the 103 symbols"></img> | Yellow circle *p6* represents the output vector. |
| <img src="/../../img/project_tokenization_repair/legend_LSTM.png" title="Long Short-Term Memory cell"></img> | The blue box always stands for the same LSTM cell. It gets an input vector and a hidden state as input and outputs a vector of the same dimension as the hidden state. |
| <img src="/../../img/project_tokenization_repair/legend_FCN.png" title="fully connected network"></img> | The green box represents the fully connected network with 103 output neurons. |
| **Arrows** | Arrows visualise the information flow. Outgoing arrows are outputs and incoming arrows are inputs of the network's submodules. |

The following image shows the computational graph of the unrolled forward model operating on the example sequence *Hello world.* and predicting the character following the prefix *Hello*.

<img src="/../../img/project_tokenization_repair/unidirectional.png" title="The unidirectional forward model predicting the probability distribution for the character following 'Hello'."></img>

The whole sequence can be processed in a single run of the LSTM cell over the sequence, feeding all hidden states into the fully connected part of the network, as shown in the following image.

<img src="/../../img/project_tokenization_repair/unidirectional_all.png" title="The unidirectional forward model predicting probability distributions at all positions in a single run."></img>

Drawing the computational graph of the backward model is left as a task to the reader (hint: only a few changes have to be made regarding the beginning and end of the sequence and the operational direction of the LSTM cell).

## Combined models

The forward and backward models estimate probability distributions \\(p\_{\rightarrow}(c|\mathit{prefix})\\) and \\(p\_{\leftarrow}(c|\mathit{suffix})\\) at each position in the sequence independent from each other.
The probability distributions estimated by the two models can be combined into a single probability distribution \\(p(c|\mathit{prefix}, \mathit{suffix})\\), thereby using the information given by the whole sequence.

The combination can be done by multiplying the probabilities estimated by the two models (equivalent to a logical conjunction AND) or by averaging them (equivalent to a logical disjunction OR).

In my experiments, the conjunction method was superior, and was improved by not only estimating the probability for a space insertion or deletion, but comparing it with the probability for the characters preceeding and following the space.
<details>
<summary>DETAILS: different combination methods for character prediction, space insertion and space deletion.</summary>

### Character prediction

To predict the most likely character to occur between a given prefix and suffix, the forward model is fed with the prefix and the backward model with the suffix.
Then, the estimated probabilites can be combined using a logical conjunction (AND) or a logical disjunction (OR).

The conjunction is computed by multiplying the probabilities estimated by the two models and normalizing over all characters. Since for prediction we are only interested in the most likely character, we can drop the normalization term (which is the same for all characters).
I denote the probabilities estimated by the forward model with an arrow pointing in reading direction, and reversely for the backward model.

$$\mathit{p_{conjunction}}(c|\mathit{prefix}, \mathit{suffix}) = p\_{\rightarrow}(c|\mathit{prefix}) \cdot p\_{\leftarrow}(c|\mathit{suffix})$$

The disjunction is similar, but uses the average instead of the product.

$$\mathit{p_{disjunction}}(c|\mathit{prefix}, \mathit{suffix}) = \frac{ p\_{\rightarrow}(c|\mathit{prefix}) + p\_{\leftarrow}(c|\mathit{suffix}) }{2}$$

### Delimiter insertion

I designed different methods to combine the output of the forward and backward model for estimating the probability of a space insertion operation.
As an example consider the input sequence *Helloworld.* and assume we want to compute the probability for a space between *Hello* and *world.*.

#### Conjunction and disjunction

Like above, we can compute the logical conjunction and disjunction by multiplying and averaging the forward and backward probabilities for a space at the given position.

$$\mathit{p\_{insert}^{conjunction}}(\mathit{prefix}, \mathit{suffix}) = p\_{\rightarrow}(\mathit{space}|\mathit{prefix}) \cdot p\_{\leftarrow}(\mathit{space}|\mathit{suffix})$$

$$\mathit{p\_{insert}^{disjunction}}(\mathit{prefix}, \mathit{suffix}) = \frac{ p\_{\rightarrow}(\mathit{space}|\mathit{prefix}) + p\_{\leftarrow}(\mathit{space}|\mathit{suffix})}{2}$$

#### Normalized conjunction

We can normalize the combined values to create a probability distribution.

$$\mathit{p\_{insert}^{normalized}}(\mathit{prefix}, \mathit{suffix}) = \frac{ p\_\rightarrow(\mathit{space}|\mathit{prefix}) \cdot p\_\leftarrow(\mathit{space}|\mathit{suffix})}
 {\sum\_{c \in \mathcal{A}}  p\_\rightarrow(c|\mathit{prefix}) \cdot p\_\leftarrow(c|\mathit{suffix})}$$

Note that normalization does not affect the disjunction, since here the normalization term is always equal to one.

<details>
<summary>DETAILS: proof.</summary>

To show: the disjunction probabilities of all characters in the character set \\(\mathcal{A}\\) sum up to one.<br>
Proof:

$$\sum_{c \in \mathcal{A}} \mathit{p\_{disjunction}}(c|\mathit{prefix}, \mathit{suffix})$$

$$= \sum_{c \in \mathcal{A}} \frac{ p\_{\rightarrow}(c|\mathit{prefix}) + p\_{\leftarrow}(c|\mathit{suffix}) }{2}$$

$$= \frac{1}{2} \Big( \sum\_{c \in \mathcal{A}} p\_{\rightarrow}(c|\mathit{prefix}) + \sum\_{c \in \mathcal{A}} p\_{\leftarrow}(c|\mathit{suffix}) \Big)$$

$$= \frac{1}{2} \cdot (1 + 1)$$

$$= 1 $$
</details>

#### Comparison with the given sequence

Instead of estimating the space insertion probability just based on the forward and backward probabilities of a space character, we can compare it with the probabilities of the original sequence's characters.

<details>
<summary>DETAILS: intuition for the comparison method.</summary>

This is in particular useful in cases like the token *into*, where the probability for a space after *in* and before *to* is high (resulting in a high insertion probability), although the token *into* makes perfectly sense.
Considering the *t* after *in* and the *n* before *to*, the model should lower the insertion probability, because the token *into* appears frequently in the training data.
</details>

First, the probabilities of a space and of the original sequence are estimated using the conjunction or disjunction method, and then compared to give a score for the insertion operation.
The methods *first* and *last* return the first or last character of the prefix or suffix.

$$\mathit{p\_{space}^{conjunction}}(\mathit{prefix}, \mathit{suffix}) = p\_{\rightarrow}(\mathit{space}|\mathit{prefix}) \cdot p\_{\leftarrow}(\mathit{space}|\mathit{suffix})$$
$$\mathit{p\_{sequence}^{conjunction}}(\mathit{prefix}, \mathit{suffix}) = p\_{\rightarrow}(\mathit{first}(\mathit{suffix})|\mathit{prefix}) \cdot p\_{\leftarrow}(\mathit{last}(\mathit{prefix})|\mathit{suffix})$$
$$\mathit{p\_{insert}^{conjunction, compared}}(\mathit{prefix}, \mathit{suffix}) = \frac{ \mathit{p\_{space}^{conjunction}}(\mathit{prefix}, \mathit{suffix}) }{ \mathit{p\_{space}^{conjunction}}(\mathit{prefix}, \mathit{suffix}) + \mathit{p\_{sequence}^{conjunction}}(\mathit{prefix}, \mathit{suffix}) }$$

(The formulas for the disjunction are analogue.)

### Delimiter deletion

The methods from above can also be used to estimate the probability that a space has to be deleted.
In that case, 1 minus the space probability is the estimate for the deletion probability.

For example, assume that we are given the input sequence *Hell o!* and want to estimate the probability that the space between *l* and *o* has to be deleted.

$$\mathit{p\_{delete}^{conjunction}}(\mathit{prefix}, \mathit{suffix}) = 1 - p\_{\rightarrow}(\mathit{space}|\mathit{prefix}) \cdot p\_{\leftarrow}(\mathit{space}|\mathit{suffix})$$

$$\mathit{p\_{delete}^{disjunction}}(\mathit{prefix}, \mathit{suffix}) = 1 - \frac{ p\_{\rightarrow}(\mathit{space}|\mathit{prefix}) + p\_{\leftarrow}(\mathit{space}|\mathit{suffix}) }{2}$$

$$\mathit{p\_{delete}^{normalized}}(\mathit{prefix}, \mathit{suffix}) = 1 - \frac{ p\_\rightarrow(\mathit{space}|\mathit{prefix}) \cdot p\_\leftarrow(\mathit{space}|\mathit{suffix})}
 {\sum\_{c \in \mathcal{A}}  p\_\rightarrow(c|\mathit{prefix}) \cdot p\_\leftarrow(c|\mathit{suffix})}$$

For the comparison with the original sequence, the space and sequence probabilities are computed like above, and then the deletion probability becomes the following:

$$\mathit{p\_{delete}^{conjunction, compared}}(\mathit{prefix}, \mathit{suffix}) = \frac{ \mathit{p\_{sequence}^{conjunction}}(\mathit{prefix}, \mathit{suffix}) }{ \mathit{p\_{space}^{conjunction}}(\mathit{prefix}, \mathit{suffix}) + \mathit{p\_{sequence}^{conjunction}}(\mathit{prefix}, \mathit{suffix}) }$$

(Analogue for the disjunction.)
</details>

## Bidirectional models

So far we have used two completely independent models processing the input sequence forward and backward, and combined their probability estimations externally. 
It is a limitation of this approach that one model solely uses the information given by the prefix and the other solely uses the suffix information.
A more sophisticated model processes the input sequence in both directions to generate hidden states, and then combines the hidden states internally to generate a character probability distribution.

### Bidirectional LSTM

The computational graph of this approach is shown in the following scheme, where the input sequence is *Hello world.* and the model estimates a probability distribution over all characters between the prefix *Hello* and the suffix *world.*.
In addition to the forward LSTM cell another LSTM cell is introduced, that traverses the sequence in backward direction.
The hidden states of both LSTM cells get concatenated into a single vector that is fed into the fully connected network, which again has a softmax output layer with 103 neurons (for the 100 characters and 3 extra symbols).

<img src="/../../img/project_tokenization_repair/bidirectional.png" title="The bidirectional model predicting a character probability distribution."></img>

This computational graph is used to predict characters and to train the model, as well as to estimate the probability that a space has to be deleted (which is the inverse of the space probability).
$$\mathit{p\_{delete}^{bidirectional}}(\mathit{prefix}, \mathit{suffix}) = 1 - p\_{\leftrightarrow}(\mathit{space}|\mathit{prefix}, \mathit{suffix})$$

### Insertion probabilities

In order to estimate insertion probabilities, e.g. for an insertion between the prefix *Hell* and the suffix *o world.*, the same LSTM cells and fully connected network are used with a slight change of the connections between the LSTM cells and the FCN:

<img src="/../../img/project_tokenization_repair/bidirectional_insertion.png" title="The bidirectional model in insertion mode."></img>

(In comparison to the picture above, there is no gap left regarding the input sequence.)

### Sigmoidal output layer

The softmax output layer constrains the model to output probabilities that sum up to 1.
This can be problematic in cases where no character at all shall be inserted.
Since the model has not been trained with such examples, the output could be arbitrary.
We can hope that this arbitrariness looks sufficiently different from a meaningful prediction, such that we can distinguish between them.
Alternatively we can drop the probability distribution constraint by replacing the softmax output layer with a sigmoidal output layer, therefore allowing the network to output probability values that are all close to zero when no character has to be inserted.
The model with sigmoidal output layer is (in addition to the generative training) also trained with negative examples, that is, using the scheme shown for the insertion probabilities and all-zero vectors as target outputs.

## Sequence length and batching

Due to the recurrence scheme of the LSTM cell, the same neural models can process sequences of varying lengths.
When fed with a sequence of length *n*, the network gets unrolled for *n* steps (that is, the same weight matrices and operations are used *n* times).
To accelerate training, a batch of multiple sequences is fed into the network together, which requires sequences of the same batch to be equally long.
To group sequences of the same length into batches, the training data sets were divided into files with sequences of the same length, which were then split into batches. The batches were shuffled before training.

## Architecture and hyperparameter selection

I experimented with different network architectures and selected one that gave a good tradeoff between training time and character prediction accuracy.
The chosen forward and backward model architecture consists of a single-layer LSTM cell with 512 neurons and one fully connected hidden layer with 512 neurons.
In order to compare models of equal complexity, the same parameters were used for the bidirectional models (having two LSTM cells instead of one).
<details>
<summary>DETAILS: experiments and more hyperparameters.</summary>

I compared several architectures for the unidirectional forward model.
The LSTM cells of the models contained one or two layers with either 256, 512 or 1024 neurons, and the fully connected networks had one hidden layer with 256 or 512 neurons.
The models were trained on the Europarl training data set and their character prediction accuracy evaluated on the validation data set.
I then decided for the architecture with one layer with 512 neurons for the LSTM cell and one hidden layer with 512 neurons for the fully connected network, because it had the nicest tradeoff between training time (43 minutes per epoch on the Europarl training set using a GeForce GTX 1060 GPU) and character accuracy (73.99% on the validation set after one epoch).

The same architecture was used for the backward model and the bidirectional model (the latter having two LSTM cells instead of one), in order to compare models of equal complexity.

For the fully connected hidden layers the rectified linear unit activation function (ReLU) was used, and all models were trained with the Adam optimizer.
The batch size was set to 128.

I experimented with introducing dropout layers with a dropout rate of 0.5 during training, but the results did not improve.

The models used for the final evaluation were trained on half of the Wikipedia data set.
</details>

## Greedy algorithm and decision threshold fitting

The greedy Tokenization Repair algorithm evaluates all insertion probabilities and all deletion probabilities, filters for the operations with probabilities above a threshold, translates them into scores using a linear transformation, and applies the operation with the highest score.
All probabilities are re-evaluated before the next operation is chosen.
The algorithm stops when all probabilities are below the threshold.
<details>
<summary>DETAILS: linear transformation.</summary>

The linear transformation is done to be able to compare insertion and deletion probabilities, which can follow different distributions and have different decision thresholds.
For a threshold *t* and probability *p*, the score *s* is computed as:
$$s = \frac{p - t}{1 - t}$$
</details>
<details>
<summary>DETAILS: threshold fitting.</summary>

For each of the methods described above, two decision thresholds are fit on the validation data set: one for classifying inserted delimiters and one for deleted delimiters.

The following figure gives an example of how precision, recall and F1 vary with different thresholds between 0.1 and 1 for the bidirectional sigmoidal model predicting inserted spaces.
The threshold that optimizes the F1 score is indicated with a small black circle.

<img src="/../../img/project_tokenization_repair/threshold_fitting.png" title="Precision, recall and F1 vary with different decision thresholds. The best threshold is fit on the validation data set." width="70%"></img>

</details>

# <a id="language_model_evaluation"></a> Language Model Evaluation

This section gives an evaluation of the character-based language models on the task they are trained on: the prediction of a character given the rest of the sequence.

The following table gives the top 1 and top 5 accuracy<a href="#footnote_accuracy">¹</a>
for the unidirectional models, the best combined model and the two bidirectional models, as well as precision, recall and F1 when only considering the prediction of space characters<a href="#footnote_f1">²</a>.
The models were evaluated on the Wikipedia test data set.

| __method__                  | __top 1 character accuracy__ | __top 5 character accuracy__ | __space precision__ | __space recall__ | __space F1__ |
| --- | --- | --- | --- | --- | --- |
| forward model           | 0.6580                   | 0.8838                   | 0.8243          | 0.9157       | 0.8676   |
| backward model          | 0.6217                   | 0.8703                   | 0.9094          | 0.9518       | 0.9301   |
| combined (conjunction)  | 0.8612                   | 0.9703                   | 0.9688          | 0.9958       | 0.9821   |
| bidirectional softmax   | 0.9500                   | 0.9875                   | 0.9918          | 0.9964       | 0.9941   |
| bidirectional sigmoidal | 0.9442                   | 0.9845                   | 0.9896          | 0.9968       | 0.9932   |

I find that all models perform better on the prediction of spaces than on the prediction of arbitrary characters.

The combination of the forward and backward model gives a big increase in performance compared to the unidirectional models.
The bidirectional models outperform the other approaches.

<a id="footnote_accuracy"></a>
¹ Top 1 and top 5 accuracy: how often is the true character the one with the highest estimated probability, and how often is it among the five characters with the highest estimated probability.<br>
<a id="footnote_f1"></a>
² Precision: when the model predicts a space, how often is it correct. Recall: when the true character is a space, how often does the model predict a space. F1: the harmonic mean of precision and recall.

# <a id="tokenization_repair_evaluation"></a> Tokenization Repair Evaluation

This section gives an evaluation of the Tokenization Repair methods on the Wikipedia test data set, where corrupted sequences were created with the error model described above and *p* set to 0.1.

The evaluated methods are:

* Google: the Google spellchecker.
* LibreOffice: the LibreOffice spellchecker.
* Aspell: the Aspell spellchecker.
* baseline: the dictionary-based dynamic programming approach with rule-based postprocessing. Varying dictionary sizes were tested on the development set, among which a dictionary with 1,000,000 gave the best results.
* baseline+RF: like baseline, but in addition with ambiguity resolving postprocessing using a Random Forest model.
* combined: the best method combining the forward and backward model (which is using the conjunction method and comparing to the original sequence).
* bidirectional softmax: the bidirectional model with softmax output layer.
* bidirectional sigmoidal: the bidirectional model with sigmoidal output layer.

| __method__                                | __precision__ | __recall__ | __F1__ | __resolved edit distance__ |
| ----------------------------------------- | ------------- | ---------- | ------ | -------------------------- |
| Google<a href="#footnote_baselines">¹</a> | 1.0000        | 0.6690     | 0.8017 | 0.6621                     |
| LibreOffice¹                              | 0.5833        | 0.2897     | 0.3871 | 0.0828                     |
| Aspell¹                                   | 0.7391        | 0.4690     | 0.5738 | 0.3034                     |
| baseline                                  | 0.8116        | 0.8036     | 0.8076 | 0.6173                     |
| baseline+RF                               | 0.8009        | 0.9334     | 0.8621 | 0.7130                     |
| combined (conjunction compared)           | 0.9489        | 0.9663     | 0.9575 | 0.9142                     |
| bidirectional softmax                     | 0.8479        | 0.8932     | 0.8699 | 0.7475                     |
| bidirectional sigmoidal                   | 0.9695        | 0.9738     | 0.9716 | 0.9462                     |

<a id="footnote_baselines"></a>
¹ Evaluated on only 20 sequences, because the evaluation is done by hand.

The Google spellchecker by far outperforms the other commercial spellcheckers.

Regarding the F1 score and fraction of resolved edit distance, my baselines can compete with the Google spellchecker.
Note that the Google spellchecker is designed for Spelling Correction rather than Tokenization Repair, and seems to be fit on human-made typos instead of random typos (for example, it does not remove spaces between numbers or before and after punctuation marks).
Also, it seems to optimize a function that weighs precision higher than recall, which might result in a non-optimal F1 score.

The ambiguity resolving postprocessing improves the baseline, increasing the F1 score from about 0.80 to about 0.86, and increasing the fraction of resolved edit distance by about 10 percent points.

The language model-based methods outperform all commercial and dictionary-based baselines.
The best method is the bidirectional model with sigmoidal output layer, which has a F1 score of about 0.97 and resolves more than 94 percent of the edit distance.

The sigmoidal output function is key to this good performance.
The bidirectional model with softmax output layer has difficulties recognizing compound words (for example, the most likely character between *care* and *fully* is indeed a space, since both are correct words - the sigmoidal output function is needed to be able to predict that actually **no** character should be present).

## Robustness to more typos

It is a relevant question whether the methods still work when more typos are present in the input sequences.
Typos affect the language models' predictions around them and might therefore prevent the methods from correcting other typos, that are close to them.

To analyse how the Tokenization Repair methods perform in presence of more typos, the parameter *p* for the error model was set to 0.5 (meaning that on average every second token gets corrupted), new decision thresholds were fit on the Wikipedia evaluation data set and the best baseline and best language model-based method from the earlier experiment were evaluated on the test data set.

| __method__                                | __precision__ | __recall__ | __F1__ | __resolved edit distance__ |
| ----------------------------------------- | ------------- | ---------- | ------ | -------------------------- |
| baseline+RF                               | 0.8635        | 0.8577     | 0.8606 | 0.7333                     |
| bidirectional sigmoidal                   | 0.9623        | 0.9574     | 0.9598 | 0.9367                     |

The dictionary-based baseline performs similar as in the experiment with less typos.

The performance of the best language model-based method drops by about one percent point in F1 score and fraction of resolved edit distance.
This suggests that the typos do indeed interfere with each other, but the interference is not strong enough to completely break the method.

# <a id="spelling"></a> Spelling Correction

Tokenization Repair is a special case of Spelling Correction, where typos only affect a small set of delimiter characters.
This section gives an analysis of whether the methods developed for Tokenization Repair can be applied to general Spelling Correction.

The error model to generate corrupted sequences was changed such that it inserts a random character from a-z, A-Z, 0-9 or a space at each position in the original sequence with probability \\( \frac{p}{2} \\) and removes each of the original characters with probability \\( \frac{p}{2} \\).
The parameter \\(p\\) now sets the character-level corruption probability (instead of token-level), and was set to 0.02 during the experiments.

Another baseline was developed which uses a dictionary containing the 1,000,000 most frequent tokens from the Wikipedia training data set.
For each token in the input sequence, that is not contained in the dictionary, the edit distance to each word in the dictionary is computed, and the token gets replaced by the word with the smallest edit distance (using the word frequency as tiebreaker when multiple words have the same edit distance).
The same is done for all possible splits of the token.
The method is sped up by limiting the edit distance to 2.

The character-based language model methods are changed such that they can not only insert spaces but all characters from the character set, and that every character in the sequence can be deleted.
Again, decision thresholds for insertion and deletion are fit on the validation data set.

The following table shows the results of the evaluation on a small subset of the test data.

| __method__                                | __precision__ | __recall__ | __F1__ | __resolved edit distance__ |
| ----------------------------------------- | ------------- | ---------- | ------ | -------------------------- |
| Google                                    | 0.7877        | 0.7231     | 0.7540 | 0.6477                     |
| baseline                                  | 0.7297        | 0.6923     | 0.7105 | 0.5233                     |
| combined (conjunction compared)           | 0.3381        | 0.4821     | 0.3975 | -0.2953                    |
| bidirectional sigmoidal                   | 0.5675        | 0.7333     | 0.6398 | 0.3316                     |

The Google spellchecker performs better than the baseline and the character-based language model methods.
However, it has difficulties with the randomly inserted typos and resolves less than two thirds of the edit distance.

The character-based language model methods do not beat the baseline.
The combined forward and backward model even introduces more new errors than it fixes.

Overall I find that the methods which work on a token level perform better than the character-based methds.
This suggests that for general Spelling Correction it might make sense to develop models that predict tokens instead of characters.

# <a id="summary"></a> Summary

Multiple methods were developed for the repair of spaces in text, which enables tokenizers to correctly parse texts containing delimiter typos.
Those methods are a dictionary-based dynamic programming algorithm, and a greedy algorithm using either a combination of two unidirectional character-based language models or a bidirectional character-based language model.

The language model methods outperform the dictionary-based algorithm and three commercial baselines.
The best method is the one that uses a bidirectional model with sigmoidal output function.
It achieves an F1 score greater than 0.97 and reduces the edit distance to the ground truth text by more than 94 percent.

The same methods were evaluated on the more general task of Spelling Correction.
Here, they do not beat a baseline that works on the token level instead of the character level.

# <a id="webapp"></a> Web Application

The project includes a web application that can be used to query the models in different ways for character prediction, sequence generation, Tokenization Repair and Spelling Correction, and visualises the results.

## Character prediction

The web application supports the use of the combined forward and backward model, the bidirectional softmax model and the bidirectional sigmoidal model to estimate character probabilities.

The following image visualises the output of the bidirectional sigmoidal model for the query *The algo rithm runsin linear time.*.
The blue dots are the probabilities for each character in the input sequence.
The orange dots are the probabilities of the most likely character at each position.
In the beginning and end of the sequence, the blue and orange dots fall together, meaning that the given characters are the most probable.
Around the typos, however, the model estimates the sequence's probability to be low.

<img src="/../../img/project_tokenization_repair/prediction.png" title=""></img>

For the same model and query, we get a visualisation of the insertion probabilities for the most likely character at each position between two characters in the input sequence.
Note how the probability for an insertion of a space between *runs* and *in* is close to 1.

<img src="/../../img/project_tokenization_repair/insertion.png" title=""></img>

For the same query, the combined model predicts the probability for each character to get deleted.
Note how the deletion probability of the space between *algo* and *rithm* is close to 1.

<img src="/../../img/project_tokenization_repair/deletion.png" title=""></img>

## Tokenization Repair

The web application supports the use of the dictionary-based baseline, the combined model, the bidirectional sigmoidal model and a mixture of the latter two (using the combined model for deletions and the bidirectional sigmoidal model for insertions) for Tokenization Repair and Spelling Correction.

In the following visualisation, the bidirectional sigmoidal method repairs the input sequence *The algo rithm runsin linear time.* to *The algorithm runs in linear time.*.
Green bars refer to insertions of spaces and red bars to deletions of spaces.
The height of each bar represents the probability the model assigns to each of the selected operations.

<img src="/../../img/project_tokenization_repair/token_correction.png" title=""></img>

## Spelling Correction

The same method corrects the input sequence *Te algorithym rruns ing liner time.* to *The algorithm runs in linear time.*.
The inserted characters are indicated on top of each green bar.

<img src="/../../img/project_tokenization_repair/spelling_correction.png" title=""></img>

## Sequence generation

The unidirectional forward model can be used to generate sequences.
For a prefix query, it repeatedly appends the most likely character to the sequence until the end-of-sequence symbol is predicted.
The character selection mode can be changed from always picking the most likely character to randomized selection.

The following image shows the model generating the most likely sequence starting with an empty prefix.
The probabilities of the three most likely characters at each position are visualised.

<img src="/../../img/project_tokenization_repair/generation.png" title=""></img>

