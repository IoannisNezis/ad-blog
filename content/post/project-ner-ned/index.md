---
title: "Named Entitiy Recognition and Disambiguation"
date: 2020-07-29T15:00:59+02:00
author: "Yi-Chun Lin"
authorAvatar: "img/ada.jpg"
tags: [NER, NED, NLP]
categories: []
image: "img/project-ner-ned/cover.png"
draft: false
---

This project uses Wikidata as the target knowledge base and aims to improve the speed and correctness of recognition and disambiguation of named entities. It is evaluated on CoNLL-2003 benchmark. A configurable framework is designed to observe the effectiveness of each part of the algorithm.
<!--more-->

# Content
- [Introduction](#intro)
- [Named Entity Recognition (NER)](#ner)
  - [POS-Tag Filter](#postag)
  - [Entity Index](#entityindex)
  - [Recognition Process](#recognition)
- [Named Entity Disambiguation (NED)](#ned)
  - [Context-Aware Weighting](#weight)
  - [Disambiguation Process](#disambiguation)
- [Configurable Features](#feature)
  - [Family Name](#familyname)
  - [Demonym](#demonym)
  - [Large Database](#largedb)
  - [Wikipedia Abstract](#wikipedia)
  - [NNP Reduction](#combine)
- [Evaluation](#evaluation)
- [Summary](#summary)

# Introduction {#intro}
Named entity recognition (NER) and named entity disambiguation (NED) are key techniques in many NLP applications, as they help computers to identify and understand the named entities in given texts. A named entity is a reference to a real-world object that can be denoted with a proper name, such as persons, locations, organizations, products, etc. For example,

- **Obama** was the president of **USA**.
- **Obama** is a city in **Japan**.

“Obama", “USA” and "Japan" are named entities. Note that "president" and "city" are not named entities, though they are indeed entities and have corresponding entries in Wikidata. In this project, we focus only on the named entities.

The task of identifying the text span of named entities in a given text is called named entity recognition (NER). In the example above, NER points out that "Obama", "USA", and "Japan" are the text span of named entities. The task of linking the text span of an named entity to the correct entry in a knowledge base is called named entity disambiguation (NED). However, a named entity doesn't always have a one-on-one mapping to the entry in the knowledge base. On one hand, a named entity can refer to different objects, like the two "Obama" in the example above. On the other hand, an object can be referred by multiple named entities, like "USA" and "United States of America" both refers to the same object (the country). This is why the task of NED is not so easy.

This project uses Wikidata as the target knowledge base and aims to improve the speed and correctness of NER and NED. The recognition process is speed up by POS-Tag filtering and a pre-generated entity index. The correctness is improved by utilizing the attributes in Wikidata, the abstracts in Wikipedia and adopting context-aware weighting. Finally, we generate a benchmark based on CoNLL-2003 and aida-yago2-dataset. A configurable framework is designed to observe the effectiveness of each part of the algorithm. A web interface is also developed to demonstrate the NER+NED engine as well as the evaluation results.

# Named Entity Recognition (NER) {#ner}
The task of named entity recognition (NER) is to locate the named entities in a given text. It is sometimes similar to Part-of-speech (POS) tagging, which is the process of determining the grammatical category of each word in the sentence. Examples of POS-tagging are shown below, with `tag` indicating the POS-tag of each word. For the complete list of tags and the meanings, see the [Penn Treebank list](https://www.ling.upenn.edu/courses/Fall_2003/ling001/penn_treebank_pos.html).

- **Obama**`NNP` was`VBD` the`DT` president`NN` of`IN` **USA**`NNP`.
- **Obama**`NNP` was`VBD` the`DT` president`NN` of`IN` **United**`NNP` **States**`NNP` **of**`IN` **America**`NNP`.

In the first sentence, all of the named entities can be easily recognized by POS-tagging, as they all have the `NNP` tag. However, doing NER by POS-tagging only works when all of the words in a named entity are `NNP`. In the second sentence, "United States of America" cannot perfectly recognized by POS-tags as "of" is not an `NNP`.

In order to recognize more named entities, especially those cannot be fully detected by POS-tagging, the ultimate way is to compare each word and its subsequences in the query sentence with the knowledge base. This process could be very time-consuming as it has an order \\(O(k^2)\\) given a text of length \\(k\\). Fortunately, improvements can be made in the following points.

### POS-Tag Filter {#postag}

The first improvement is to use the POS-tag as a filter: Only compare a word and its subsequences with the knowledge base when it has the POS-tag `NNP` or `NN`.


Since most of the words in the query sentence are not named entities, e.g. "was", "the", which can be roughly indicated by their POS-tags, there is no need to further examine these words and their subsequences in the knowledge base. Therefore, by utilizing the POS-tag as a filter, the amount of words needed to do further examination can be largely reduced to a linear scale  \\(O\\)(\\(k\\)) .

In this project, we use [spaCy](https://spacy.io) as our POS-tagger, as it is one of the stat-of-the-art tagger and performs fast. Let \\(w_p\\) denotes the word at the position \\(p\\) of the query, \\(\verb|tag|(w_p)\\) denotes the POS-tag of \\(w_p\\). The filter works as follows:

1. For the word \\(w_p\\), check \\(\verb|tag|(w_p)\\) and \\(\verb|tag|(w\_{p+1})\\).
1. If both are not `NNP` or `NN`, skip further comparison.

Note that not only the tag of the current word but also the tag of the next word are checked. This is to prevent false-filtering. Consider named entities like “My Chemical Romance” (an American punk band) or “My Neighbor Totoro” (a Japanese anime film), both of them have POS-tags of the form {`PRP$`, `NNP`, `NNP`}. If we only check the current word’s tag in the filter, these entities will be skipped and cannot be detected.

### Entity Index {#entityindex}
The second improvement is to create an entity index in advance, which goes through all the entities in the knowledge base and stores them according to their beginning word and their length.

Consider the first word "Obama" in the example "Obama was the president of United States of America". Since it is an `NNP`, we need to check itself as well as all the phrases starting with it until the end of the sentence, i.e.

- “Obama”
- “Obama was”
- "Obama was the"
- ...
- "Obama was the president of United States of America"

together 9 comparisons needed to be made. However, the entities starting with the keyword “Obama” in the knowledge base, like "Obama", "Obama Domain", "Obama On My Mind" and so on, are finite and of length 1, 2, 3, 4 or 6. That means, we don't need to compare a nine-word phrase "Obama was the president of United States of America" or other lengths that are not exist in the knowledge base. By knowing this fact, only 5 comparisons are needed. Therefore, building up such an entity index can further reduce the number of consequences needed to be checked.


The entity index is built up using entities' name and their synonyms, stored under a hierarchical structure of starting word as the first layer and length as the second layer. The value is the entity's QID, or the QIDs from all the entities having the name or synonym.

Consider a toy knowledge base with only three items, the pre-established entity index would look like the following.

| Name | QID  | Synonyms |
| ---- | ---- | ----     |
| United States of America | Q30 | USA; United States |
| Union of South Africa | Q193619 | USA |
| United Kingdom | Q145| |


```
    "United": {
        "2": {
            "United States": [Q30],
            "United Kingdom": [Q145]
        },
        "4": {
            "United States of America": [Q30]
        }
    },
    "USA": {
        "1": {
            "USA": [Q30, Q193619]
        }
    },
    "Union": {
        "4": {
          "Union of South Africa": [Q193919]
        }
    }

```
### Recognition Process {#recognition}
With the help of POS-tag filter and entity index, the entire recognition process is as follows:

1. For each word \\(w_p\\) in the query, check it with the POS-tag filter. If it is not likely to be a named entity, skip further recognition. The next word to be checked: \\(w\_{p+1}\\).

2. For a suspicious word, let \\(\verb|chunk|(p,l)\\) denotes a \\(l\\)-word phrase starting from the word \\(w_p\\) in the query.
    1. Lookup the entity index with the key \\(w_p\\), get all possible lengths \\(L\\) and corresponding entities \\(E_l\\), for all \\(l \in L\\).
    1. Check \\(\verb|chunk|(p,l)\\) in the query for all possible \\(l \in L\\). If \\(\verb|chunk|(p,l) \in E_l \\), a named entity is found.
    1. Return the named entity with the longest length \\(\hat{l}\\) and its possible QIDs. The next word to be checked: \\(w\_{p+\hat{l}}\\).


# Named Entity Disambiguation (NED) {#ned}
The task of named entity disambiguation(NED) is to link the recognized named entity to the respective item in a knowledge base. There are sometimes more than one item having the same name, so the algorithm needs to choose the most suitable one among the candidates. A straight forward approach is to choose the most popular candidate. However, the drawback is easy to see. For example, the most popular item with the name “Obama” is the president Obama. Thus, the approach always links “Obama” to the president, and will fail in the case of "Obama is a city in Japan." Therefore, more clues should be taken into consideration.

### Context-Aware Weighting {#weight}
When determining the meaning of a named entity, the context of the query usually provides great hints. A candidate item that is more related to the context is more possible to be the correct answer. This is also how human understands the meaning of texts.

To measure the relevance, the idea is to look at the overlaps between the context of the query and the "content" of candidate entities. In this project, the context of a query is represented by all words with the POS-tag `NNP` or `NN` in the query. The "content" of an entity is represented by all words in the entity's name, synonyms and description. Below shows the examples of the context of the two queries, as well as the content of the two "Obama" candidates.

| Query | Context |
| ----  | ----    |
| Obama was the president of United States of America. | Obama, president, United, States, America |
| Obama is a city in Japan. | Obama, city, Japan |


| QID | Entity Name | Synonyms | Description | Content |
| ---- | ---- | ---- | ---- | ---- |
| Q76 | Barack Obama | Obama | 44th president of the United States | Barack, Obama, 44th, president, of, the, United, States |
| Q41773 | Obama | | city in Fukui prefecture, Japan | Obama, city, in, Fukui, prefecture, Japan |

 A candidate having more overlaps with the context is considered to be more related. In the table below, we try to disambiguate "Obama" in the two queries by comparing the context with the candidates' content. In the first query, the candidate Q76 (the president) has more overlaps and thus is more related, while in the second query, the candidate Q41773 (the city) is more related.

| Entity to be disambiguated (Bolded) | Candidate | Overlaps between Context and Content |
| ---- | ---- | ---- |
| **Obama** was the president of United States of America.| Q76 |  Obama, president, United, States |
| | Q41773 | Obama |
| **Obama** is a city in Japan. | Q76 | Obama |
|  | Q41773 | Obama, city, Japan |

The example is well designed to demonstrate the concept. In real cases, there could be no overlaps at all, or the amount of overlaps may not be positively correlated. Therefore, we make the disambiguation considering not only the relevance of the candidate, but also its popularity.

### Disambiguation Process {#disambiguation}

For a recognized named entity, given all its possible candidates, disambiguate by choosing the candidate with the highest score, where
$$\verb|score = popularity score \+ relevance score|$$
The popularity score comes from the entity's property *sitelinks* in Wikidata. It is an integer in the range from 0 to 367. A higher number of sitelinks indicates a more popular entity. The relevance score is computed by the number of overlaps times a weight. The weight is chosen such that about 2 to 3 overlaps can beat a very popular item. It is default to 200. In a longer query, the context may contain more words but be less representative. Therefore, if there are more than 10 words in the context, decrease the weight to 150.

Similarly, the weight should also be proportional to the description length. However, the description in Wikidata tend to be short: 96% of the description are less than 10 words. In this case, a fixed weight is sufficient. Note that the behavior changes when we later introduce the [Wikipedia abstract](#wikipedia).


# Configurable Features {#feature}
During the process of error analysis, we further improve the correctness by either utilizing more information from the knowledge base or reducing the false detected named entities. Five improvements are introduced below. They are implemented as configurable features and their effectiveness can be seen in the [evaluation](#evaluation) section.

### Family Name {#familyname}
Sometimes the query doesn't contain the complete name when mentioning a person. Consider the example "**Armstrong** may not be the first on the **Moon**." It is not possible to correctly disambiguate "Armstrong" to the entity "Neil Armstrong", because "Neil Armstrong" doesn't match the query and thus is not considered as a candidate.  This can be solved by adding its family name "Armstrong" to its synonym.

Rule: If an entity is of type "person" and has the attribute "Family Name", add its family name to its synonym.

### Demonym {#demonym}
In the benchmark CoNLL-2003, there are many entities that is loosely pointed to their country. For example, "a **South African** couple" is disambiguated to "**South Africa**"; "two **Chinese** cities" is disambiguated to "**China**". Again, it is not possible to correctly disambiguate this type of entities with our algorithm, because the texts doesn't match. But it can be solved by the attribute "Demonym", which means the people who live in the place. This is practical as the demonym itself may not be an entity in the knowledge base. Even the demonym is an entity, it is still good to link the demonym to its country for better understanding.

Rule: If the item is a country and has the attribute "Demonym", add its demonym to its synonym.

### Large Database {#largedb}
In the beginning, the Wikidata database we used was obtained from the course "Information Retrieval". It is an extracted version of Wikidata and leads to recognition limits as not all entities are included. Therefore, it is reasonable to try the full version of Wikidata database and to compare the performance difference.

### Wikipedia Abstract {#wikipedia}
When disambiguation, we compute the relevance score of a candidate from its description in Wikidata. But sometimes the description in Wikidata contains too little information and the disambiguate falls back to only depends on the popularity score. The abstract paragraph in the corresponding Wikipedia page would be a good idea to give more informative details.

Consider the query sentence "**Armstrong** was stripped of all seven **Tour de France** titles.". Since the description of the entity "Lance Armstrong" only states "American cyclist", it can't contribute any relevance score. But the term "Tour de France" appears in its Wikipedia abstract, hence can contribute to relevance score and leads to correct disambiguation results.

Note that the length of Wikipedia abstract differs in a large range. Thus, the weight is further adjusted by the length of the "content". In this project, the weight is inversely proportional to the logarithm of the length.

### NNP Reduction {#combine}
One sort of error comes from the false-recognized named entities. Specifically, it is due to the limit of the dataset that the entire entity doesn't exist but each word the entity exist. For example, "Bank Duta" is an Indonesia bank, which is not an entry on Wikidata. Meanwhile, "Bank" and "Duta" are entities on Wikidata thus results in two false-recognized named entities. Another example is "Juan Guillermo Londono", who is a Colombian journalist, which is false-recognized as "Juan" "Guillermo" "Londono". It is less possible to have single-word consequent named entities, especially when they are not so popular or related. Here, we take an easy approach to exclude consequent single-word NNP, except there is a famous NNP included.

# Evaluation {#evaluation}

We generate a benchmark to evaluate the correctness of the proposed NER+NED algorithm. The benchmark is based on the CoNLL-2003 and aida-yago2-dataset. CoNLL-2003 is an entity recognition task provided by the Conference on Computational Natural Language Learning. It includes 1,393 English and 909 German news articles with entity information. Entities are recognized and annotated by categories (LOC, ORG, PER, or MISC). Based on that, the Max Planck Institut Informatik then generated the aida-yago2-dataset, which provides YAGO2, Freebase, and Wikipedia URL annotations on the entities recognized in the CoNLL-2003. In our benchmark, we focus on the 1,393 English news articles and further translate the annotations from the aida-yago2-dataset into Wikidata annotations using mapping tables provided by the chair.


The evaluation matrix are Micro-F1 and Macro-F1 scores. Micro-F1 treats the entire dataset as a whole. It accumulates `true-positive`, `false-positive`, `false-negative` entities from all articles and then computes one F1 score. On the other hand, Macro-F1 focuses on the performance of individual article. It computes one F1 score for each article and then averages them to get the final F1 score.

Given an entity in the ground truth, we define `true-positive` when the algorithm outputs an entity that has the same starting position, length and Wikidata QID with the ground truth. We define `false-negative` if the algorithm doesn't output such an entity. We define `false-positive` if the algorithm outputs an entity covering words which don't belong to any entity in the ground truth.

Note that the evaluation matrix only takes the entities in our database, Wikidata, into consideration. That means, if an entity in the ground truth doesn't have a valid annotation in Wikidata, we ignore the algorithm's output on that entity. There are two reasons why some entities don't exist in our database: 1) They are left blank in aida-yago2-dataset because the annotator can't find a proper annotation. 2) They can't be translated from Freebase and/or Wikipedia URL to Wikidata.

![evaluation results](/../../img/project-ner-ned/ner_ned_eval_result.png)

Above shows the evaluation results of our algorithm in different configurations. Each row is a configuration with some features enabled, which is indicated by the black dot on the left-hand side. For example, the first row is the basic algorithm without any feature, and the last row is with all the features enabled. On the right-hand side shows the Micro and Macro F1 score, as well as the count of total tp, fp, fn entities. We can see not only how each feature contribute to the performance but also how different combination of features perform. Here are some observations:

1. "Family Name" and "Demonym" successfully reduce the false-negative counts

    With only "Family Name" enabled, it reduces 875 false-negative entities, and with only "Demonym" 1759 false-negative entities. These two features provide more matching possibilities on certain entities, thus make them from unrecognized to correctly recognized and disambiguated. Both features also don't introduce too many new false-positive entities. Therefore, they are very effective on this dataset.

2. "NNP Reduction" successfully reduces the false-positive counts

    With "NNP Reduction" (Combine NNPs) we reduce 1744 false-positive entities, but also introduce around 200 new false-negative entities. This is understandable as it may combine consequent correct entities into one wrong entity. Comparing the advantage to the disadvantage, it is still effective on this dataset.

3. "Wikipedia Abstracts" and "Large Database" result in minor improvements

    We can see that "Wikipedia Abstracts" and "Large Database" don't have much positive effect on the performance. When both enabled, they reduce 819 false-negative entities but also bring 528 new false-positive entities. They bring more entities and information in the recognition and disambiguation, therefore more entities are recognized, some are correct, some are wrong. We can see from the last two row that they still arise the F1 score. However, they require much more memory and processing time. Therefore, they are not that necessary on this dataset, especially when the resources are limited. Also, the algorithm may need more fine tune on the weighting to mostly gain from these two features.

# Summary {#summary}

This project uses Wikidata as the target knowledge base and aims to improve the speed and correctness of recognition and disambiguation of named entities. In the recognition stage, we use POS-Tag filtering and a pre-generated entity index to speed up the process. The recognition rate is increased by utilizing the attributes "Family Name" and "Demonym" in Wikidata, as well as reducing false-recognized named entities. In the disambiguation stage, we make use of the abstracts in Wikipedia and adopt context-aware weighting to improve the disambiguation correctness. Finally, we generate a benchmark based on CoNLL-2003 and aida-yago2-dataset. The performance of the entire algorithm and the contribution of each above mentioned feature can be viewed separately. This is also useful when later we evaluate on different benchmark. As each benchmark has different characteristics, the algorithm may need to be adjusted separately.
