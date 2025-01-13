---
title: "Wikipedia Entity Linking and Coreference Resolution in C++"
date: 2022-06-24T12:05:46+02:00
author: "Benjamin Dietrich"
authorAvatar: "img/project_wikipedia_entity_linker_and_coreference_resolution_cpp/benjamin_dietrich.jpeg"
tags: [nlp, entity linking, coreference resolution, C++, Wikipedia]
categories: [project]
image: "img/project_wikipedia_entity_linker_and_coreference_resolution_cpp/cover.png"
draft: false
---
Implementing a C++ version of an already existing Python based entity linking and coreference resolution system for Wikipedia was what this project was aiming for. The goal was to improve the runtime of the system while maintaining the already good linking results. How this goal was achieved will be discussed in this article.

<!--more-->

## Content
1. [Introduction](#introduction)
    - [Entity Linking](#entity-linking)
    - [Coreference Resolution](#coreference-resolution)
    - [Natural Language Processing](#natural-language-processing)
2. [Project](#project)
    - [Scope](#scope)
    - [Goal](#goal)
3. [Implementation of the Linking Process](#implementation-of-the-linking-process)
4. [New Features](#new-features)
    - [Prefix tree for Entity Searching](#prefix-tree-for-entity-searching)
    - [Text as Bitmap](#text-as-bitmap)
    - [Bit arithmetic for Unicode decoding](#bit-arithmetic-for-unicode-decoding)
5. [Linking Results and Performance](#linking-results-and-performance)
6. [Conclusion](#conclusion)
    - [Ideas to improve the Linking Results](#ideas-to-improve-the-linking-results)
    - [Ideas to improve the Performance](#ideas-to-improve-the-poerformance)

## Introduction
### Entity Linking
In general, an entity describes a person, a real object or an abstract object. What characterizes the entity in each case is that it is uniquely identifiable. In entity linking, we identify entities contained in texts by their name. For example, the sentence *Albert Einstein was born in Ulm* would contain the entities *Albert Einstein* and *Ulm*.

![entity_linking](/../../img/project_wikipedia_entity_linker_and_coreference_resolution_cpp/entity_linking.png)

The goal of entity linking is to recognize and link named entities in a text to the corresponding entities from the knowledge base. If the entity linking succeeds with the knowledge base, it is stored as a pair of the text position and a unique identification number. Such a pair is also called *Entity Mention*. Our knowledge base comes from Wikidata and contains about 6 million entities. The unique identification number in the Wikidata knowledge base is called *Q-ID*. The knowledge base does not only contain the names of the entities, but also for example aliases, gender or types of these entities.

Entity linking is used wherever the content of a text needs to be extracted and understood. This would be the case, for example, with chatbots, text analysis systems or recommendation systems.

### Coreference Resolution
A coreference is a phenomenon in linguistics where different expressions are used for the same object. In coreference resolution, we try to recognize these expressions and assign them to the referenced entities. An example of a coreference is in the following two sentences:

![pronoun](/../../img/project_wikipedia_entity_linker_and_coreference_resolution_cpp/pronoun.png)

Here the entity is *Marie Curie* and the corresponding coreference is the pronoun *She* from the second sentence. It refers to 'Marie Curie' as well. Coreference resolution is of high importance for the field of Natural Language Processing, because it is very important to understand the context of a text correctly.

### Natural Language Processing
An NLP library analyzes and processes a text with algorithms and artificial intelligence in such a way that detailed information about the words contained is then available.

The NLP library used in this project is called spaCy. This library is very powerful and is already used in the original version of the Wiki Entity Linker and is based on Python. This project uses a wrapper that makes spaCy available in C++ as well. We did a long research and looked for an alternative based on C++. Most of them were already outdated or not free and usable. Therefore, the decision was made to use spaCy for the time being. It has among many other things Named Entity Recognition (NER). With this, spaCy recognizes which token in a text belong to an entity. However, spaCy cannot recognize which entity it is exactly. In addition, labels are assigned to the entities, which for example provide information about whether the entity is a person, an organization or an event. Also, spaCy has tokenization, part-of-speech tagging, and dependency parsing, among other features. This additionally allows the meaning of a word in a sentence to be identified and what the word refers to. These functions make it possible to analyze the text of an article and use the information to detect entities or link coreferences.

## Project
At the Chair of Algorithms and Data Structures the [Wiki Entity Linker](https://github.com/ad-freiburg/wiki_entity_linker/blob/master/README.md) was developed in Python by [Natalie Prange](https://github.com/flackbash) and [Matthias Hertel](https://github.com/hertelm). It is able to link any number of Wikipedia articles and provides excellent linking results. The disadvantage of the system is the long processing time. It results from the overhead Python brings with it and can take several days to link an entire Wikipedia dump. In order to shorten the processing time, this project should now translate the Wiki Entity Linker from Python to C++.

### Scope
The Wiki Entity Linker consists of basically four modules:
  - **Parser:**
      Prepares the text for the linking system.
  - **Link Text Linker:**
      Links the previously found hyperlinks.
  - **Popular Entities Linker:**
      Searches for entities in the text not previously recognized by hyperlinks.
  - **Coreference Resolution System:**
      Searches the text for entity types and pronouns.

### Goal
The clear goal of the project was to implement a basic version of the already existing system. The new system should maintain the results of the existing system while reducing the runtime. There are some special linking rules which are not planned to be implemented in this first version. Besides that, it should deliver the same quality as the original version.

## Implementation
### Linking Process
The system was implemented modularly and has the mentioned system [modules](#scope). It goes through seven steps from reading the data to outputting the linked articles.

![System](/../../img/project_wikipedia_entity_linker_and_coreference_resolution_cpp/system.png)

**1. Loading the knowledge base**
First, the knowledge base, which consists of several TSV files, is loaded and stored in unordered maps to be used later for entity mapping. This process is triggered by the parser and since it takes a long time, rudimentary multithreading has already been implemented to reduce the processing time somewhat.

**2. Text parsing:**
The text of the article is loaded and parsed. HTML tags are removed from the text and replaced by the respective text. Hyperlinks contained as HTML tags are saved for later processing. Both the hyperlink target and the hyperlink text are relevant for linking, which is why both are stored. The hyperlink target is the actual link to the entity's corresponding Wikipedia article, and the hyperlink text is a string representing the hyperlink in the text.

**3. Linking System:**
There are three linkers included in the linking system. The Link Text Linker, the Popular Entities Linker and the Coreference Resolution System. The Linking System itself takes control of the three linkers.

**4. Link Text Linker**
The linking system first starts the Link Text Linker which processes the hyperlinks previously discovered by the parser. For each hyperlink, it searches the corresponding identification number from the knowledge base and stores the result as an entity mention. In addition, the Link Text Linker loads any aliases of the entity from the knowledge base based on the Q-ID found. Once the Link Text Linker is done, it returns the found information to the Linking System. The Linking System now has a set of search terms containing the entities and aliases. Now the Linking System searches the whole text from beginning to end for these search terms. To do this efficiently it uses a prefix tree which is explained in more detail below. Each time a search term is found in the text, it is linked to the corresponding entity and stored as an entity mention.

**5. Popular Entities Linker:**
Now the Linking System starts the Popular Entities Linker which uses spaCy to search for additional entities in the text which have not yet been detected by the Link Text Linker. If an entity is detected which has not been linked yet, the Popular Entities Linker checks if an alias or label with the same name exists in the knowledge base. If both an alias and a label are found, the entity with the higher site link count is preferred. The site link count is the number of languages a Wikipedia article has been translated into and thus reflects the popularity of the entity. In the same way as in the Link Text Linker all search terms are stored in the prefix tree and the Linking System searches and links the corresponding entities.

**6. Coreference Resolution System**
Finally the Linking System starts the Coreference Resolution System which has the task to link pronouns and entity types. Let's assume that we have the following two sentences:

![corefrence resolution system](/../../img/project_wikipedia_entity_linker_and_coreference_resolution_cpp/crs.png)

The named entity in the sentence is *MSY Wind Surf* and its type is *cruise ship*. Beside the pronouns, the Coreference Resolution System also searches and links types like this.

The Coreference Resolution System uses spaCy to traverse the text token by token. The system checks if the token is a pronoun, a part of an entity or part of a type. Types by our definition are always introduced by *the*. The Coreference Resolution System links all found pronouns to the last found entity with is matching gender, if the pronoun has a gender. Also, it links the types to the corresponding entities based on the knowledge base.

**7. Output:**
Finally, all entity mentions are output together with the corresponding article in a JSONL file.

## New features
### Prefix tree for entity searching
To search for entities and their aliases in the text, the system uses a prefix tree as a data structure. This allows to efficiently check if a word or a letter is part of a searched string. Two variants of a prefix tree were implemented for the project. One reads letters and checks for each letter whether it is part of a sequence of letters that make up an entity. The second variant does the same, only with a whole word in each node instead of a letter. Both are used in different places in the system, for example when searching for further occurrences of already found entities in the text or when searching for entity types in the Coreference Resolution System.
The reason we need both is, that we are not searching the text word by word in every linker. That is because some letters are stored in Unicode format, which makes indexing a found entity more complicated. This is described in more detail in the section [Bit arithmetic for Unicode decoding](#bit-arithmetic-for-unicode-decoding).

The prefix tree is first filled with strings that are to be recognized later. Then a text is searched for these stings successively. Each part of the text is passed into the prefix tree, where it checks whether one of the currently running search threads can reach a matching next node. A search thread is a new search that always starts with a none alphabetic character in the text and begins at the root node. If a thread does not find a next node that matches the searched text part, it is terminated. If a thread encounters an endpoint, it signals that by returning parameters that are used to then link the entity.

For example, lets assume that we want to search for the strings *say*, *sad*, *i'm*, *dad* and *dot*. At first, we add the strings to the prefix tree one after the other. The strings will be added letter by letter in this case. While adding a string, the prefix tree checks if there already exists a node with the letter in it. If this is the case, the tree doesn't create a new node, it just goes to that existing node and proceeds this process, till there is no matching node anymore. After that, it creates new nodes for the rest of the string if there is still something left. In the example below, you can see that for *say* and *sad* have the same prefix *sa* and thus share the same nodes for that part.

![Trie](/../../img/project_wikipedia_entity_linker_and_coreference_resolution_cpp/trie.png)

Now let's try to search for the strings we saved in the prefix tree. We are searching in the sentence: *"I'm on a bicycle tour with my dad."* We start at the letter *I* and with a new thread in the tree. The tree checks if there is a node which can be reached from the current node. At the moment, the current node is the root node. In fact there is a node with the letter *I* so the thread changes from the root node to the node with the letter *I*. Now the next character will be checked, which is the apostrophe. Again we find a node, and for the next letter *m* we do find one too.

At this point we reach an end node and the tree returns the string and it's length. With that information, we can look up its identification number in the knowledge base to link it. This process repeats itself till the complete text has been searched.

#### Text as Bitmap
Each letter of the text is stored in a bitmap and marked as false. False indicates that the letter is not yet part of a linked entity. While the first two linkers are running, the places in the bitmap that are part of a linked text passage are set to true.

![bitmap](/../../img/project_wikipedia_entity_linker_and_coreference_resolution_cpp/bitmap.png)

This allows us to skip parts of the text while searching for Entities, if they have already been linked in the past. Also, it helps to keep track of the entities which have been encountered while processing the text in the Coreference Resolution System. By our definition, two entities are separated by at least one space. This means that between two entities in the bitmap we always find a at least one false bit. While processing the text in the Coreference Resolution System, every time we encounter a token index which has been linked, we check if there has been a false bit between this token index and the previous one. If that is the case, we know this is a new already linked entity. The entity will be loaded from the already found and index wise sorted entity mentions. The loaded entity mention will then be stored in a vector with exactly four entries. The first three entries represent the genders male, female and neutral. The fourth entry always contains the latest entity encountered to the current point in the text. Every time we encounter a pronoun, we link it to the entity stored in the vector and based on the gender of the pronoun.

#### Bit arithmetic for Unicode decoding
A big challenge in the implementation was the different encoding of letters. In the knowledge base the URI format is used and in the article text Uni-Code occurs. Python automatically recognizes both URI and Uni code, but this is not the case in C++. The URI code was fairly easy to clean up, whereas the Uni code was a bit more challenging. A letter in Uni-Code consists of up to 4 so-called code points. A code point is one byte in size. Each byte is counted as one letter in a string in C++ and also increments the length of the string by one. This led to problems with the indexing of the entities, because it is important to specify the start and end point correctly in the text. In some places it was therefore necessary to work with a CPP index which is incremented with each code point in the text and a uni-code cleaned index which is only incremented if the considered code point is a leading code point. Leading code points basically indicate the beginning of a new letter. To decide whether it is a leading code point, the system takes advantage of the fact that the two most significant bits of non-leading code points are always 1 and 0.

## Linking results and performance
In order to compare the linking results of the original version, and the new C++ version the Wiki-Ex benchmark was used. It contains 80 Wikipedia articles, including hyperlinks with manually generated ground truth annotations.

![Linking Results](/../../img/project_wikipedia_entity_linker_and_coreference_resolution_cpp/linking_result.png)

- **Precision:** Indicates how many of the detected entities were also linked correctly.
- **Recall:** Indicates how many of the entities contained in the text were detected at all.
- **F1:** Balances precision and recall and is the most important measure for the quality of the system.

In terms of linking results, the C++ version already comes close to the Python version. However, there is still something missing in all three categories to be on par. This is to be expected, since some subtleties of linking have not been implemented in this project. To do this is part of the following bachelor thesis.

![Time Usage](/../../img/project_wikipedia_entity_linker_and_coreference_resolution_cpp/performance.png)

The performance has been measured by using both systems on a file with 1000 articles. Time-wise, as expected, the C++ version is significantly faster than the Python version. Especially interesting here is that the spaCy library, which is Cython based and used in both projects, takes the absolute largest part of the time in the C++version. This is also a good point for future performance improvements.

## Conclusion
The project meets the requirements and fulfills the performance expectations. Nevertheless, it still offers enough room for further improvements in terms of performance and linking results. The whole improvement process is still in a very early stage, but there are already a couple of ideas how to improve the system from here on.

### Ideas to improve the linking results
In my bachelor theses, the Coreference Resolution System will get more detailed rules to determine which entity a pronoun is referencing to. The rules will be based on grammar rules, at the moment the linking is just based on the entity which has been encountered the latest.

Also, If the full name of a entity person has been encountered, the Link Text Linker will recognize its first or last names even if the other part isn't available.

### Ideas to improve the performance
Performance will be the main topic of my upcoming bachelor theses. To archive a better runtime, I will implement multi threading in the whole system. At this point it is not clear if it would be faster, to start a thread for each article or multiple threads for the different parts of the system. I will investigate both possibilities.

Also, to overcome the processing time of spaCy, it is maybe useful to pre-process the article texts while also loading the mappings. This way they can directly be used from the Linking System when the linking process starts. Another possibility could be to experiment with alternative NLP libraries instead of spaCy.
