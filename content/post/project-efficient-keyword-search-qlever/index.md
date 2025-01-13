---
title: "Enhancing QLever's Text Search: Structure and Features"
date: 2024-05-13T16:51:20+02:00
author: "Nick Göckel"
authorAvatar: "img/ada.jpg"
tags: ["QLever", "SPARQL", "RDF", "SPARQL+Text", "Text Search", "Combined Search"]
categories: ["project"]
image: "img/designer.jpeg"
draft: false
---

The QLever search engine is a powerful tool for searching both structured and textual data simultaneously. However, the initial text search implementation in QLever had poor maintainability and readability. In this blog post, we will demonstrate how a simple change in the code structure significantly improved the code quality and facilitated the implementation of new features.

<!--more-->

# Table of Contents
1. [Introduction](#introduction)
2. [Understanding Key Concepts](#understanding-key-concepts)
    - [SPARQL](#sparql)
    - [QLever](#qlever)
3. [QLever's Combined Search Capability](#qlevers-combined-search-capability)
4. [The New Feature: Prefix Search Completion](#the-new-feature-prefix-search-completion)
5. [Improving the Text Search Structure](#improving-the-text-search-structure)
    - [Old Structure](#old-structure)
    - [New Structure](#new-structure)
    - [Feature Implementation](#feature-implementation)
6. [Conclusion](#conclusion)

## Introduction
In this blog post, we will explore the enhancements made to the QLever search engine, focusing on the restructuring of its text search functionality. We will discuss the basic concepts of SPARQL and QLever, showcase the combined search capability, and explain the newly added prefix search completion feature. Finally, we will dive into the changes in the code structure that facilitated these improvements.

## Understanding Key Concepts

### SPARQL

SPARQL is a query language similar to SQL, designed for retrieving information from RDF (Resource Description Framework) knowledge bases. Unlike SQL, which operates on relational databases, SPARQL works with RDF data, stored as triples consisting of a subject, a predicate, and an object. 

```sparql
SELECT ?subject WHERE {
  ?subject <Award_Won> <Nobel_Prize_in_Physics> .
}
```

Subject | Predicate | Object
--- | --- | ---
<Albert_Einstein> | <Award_Won> | <Nobel_Prize_in_Physics>
<Carl_Bosch> | <Award_Won> | <Nobel_Prize_in_Chemistry>
<Charles_Darwin> | <Award_Won> | <Royal_Medal>
<Marie_Curie> | <Award_Won> | <Nobel_Prize_in_Chemistry>
<Marie_Curie> | <Award_Won> | <Nobel_Prize_in_Physics>


<figure id='fig1'>
<figcaption>Figure 1: Example of a SPARQL query and an RDF database.</figcaption>
</figure>

Figure 1 illustrates a SPARQL query retrieving subjects with the predicate `<Award_Won>` and the object `<Nobel_Prize_in_Physics>`. The query returns `<Albert_Einstein>` and `<Marie_Curie>`.

### QLever

QLever is a SPARQL engine that executes queries on RDF databases. Part of what sets QLever apart from other SPARQL engines is its ability to perform combined searches, allowing simultaneous searches on structured data from an RDF knowledge graph and textual data from a text corpus.

## QLever's Combined Search Capability

QLever enables combined search queries that integrate structural and textual data. The example below demonstrates a query retrieving texts mentioning Nobel Prize-winning scientists and containing the word "astrophysics":

```sparql
# PREFIX declarations
PREFIX wd: <http://www.wikidata.org/entity/>
PREFIX wdt: <http://www.wikidata.org/prop/direct/>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX ql: <http://qlever.cs.uni-freiburg.de/builtin-functions/>
SELECT ?name ?text WHERE {
  ?scientist wdt:P166 wd:Q38104 . # wdt:P166 = "award won" and wd:Q38104 = "Nobel Prize in Physics"
  ?scientist rdfs:label ?name . # get the name behind the id
  ?text ql:contains-entity ?scientist . # magic text search predicate introduced by QLever
  ?text ql:contains-word "astrophysics" . # magic text search predicate introduced by QLever
  FILTER (LANG(?name) = "en") # filter out non-English results
}
TEXTLIMIT 2 # text search keyword introduced by QLever
```
<figure id='fig2'>
<figcaption>Figure 2: Example of a query using QLever's combined search feature.</figcaption>
</figure>

<style>
/* Work around because otherwise, the syntax highlighter messes up the colouring of the TEXTLIMIT keyword. */
#site-main > div > article > section > div > div:nth-child(17) > pre > code > span:nth-child(23) {
  color: #66d9ef !important;
}
</style>
Here we can see the first five results we get if we run the query on the Wikidata dataset:

?name | ?text
--- | ---
Vitaly Ginzburg | He also headed the Academic Department of Physics and Astrophysics Problems, which Ginzburg founded at the Moscow Institute of Physics and Technology in 1968.
Adam Riess | In astrophysics, Press is best known for his discovery, with Paul Schechter, of the Press–Schechter formalism, which predicts the distribution of masses of galaxies in the Universe; and for his work with Adam Riess and Robert Kirshner on the calibration of distant supernovas as "standard candles".
Frank Wilczek | He has worked on condensed matter physics, astrophysics, and particle physics.
Michel Mayor | Queloz studied at the University of Geneva where he subsequently obtained a MSc degree in physics in 1990, a DEA in Astronomy and Astrophysics in 1992, and a PhD degree in 1995 with Swiss astrophysicist Michel Mayor as his doctoral advisor.
Didier Queloz | Queloz studied at the University of Geneva where he subsequently obtained a MSc degree in physics in 1990, a DEA in Astronomy and Astrophysics in 1992, and a PhD degree in 1995 with Swiss astrophysicist Michel Mayor as his doctoral advisor.

If we also want to find the results for the word "astrophysicist" we can do a prefix search instead. Replacing the word "astrophysics" with "astrophy*" in the query will return results for both "astrophysics" and "astrophysicist".

## The New Feature: Prefix Search Completion
To enhance the search experience, we implemented a new feature in QLever that provides prefix search completion. This feature allows users to retrieve the completed keywords for prefix text searches, expanding the search capabilities and returning more relevant results.

To demonstrate this feature, we can modify the previous query to use a prefix search instead of an exact word match. The updated query is shown below:

```sparql
# PREFIX declarations
PREFIX wd: <http://www.wikidata.org/entity/>
PREFIX wdt: <http://www.wikidata.org/prop/direct/>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX ql: <http://qlever.cs.uni-freiburg.de/builtin-functions/>
SELECT * WHERE {
  ?scientist wdt:P166 wd:Q38104 . # wdt:P166 = "award won" and wd:Q38104 = "Nobel Prize in Physics"
  ?scientist rdfs:label ?name . # get the name behind the id
  ?text ql:contains-entity ?scientist . # magic text search predicate introduced by QLever
  ?text ql:contains-word "astrophy*" . # magic text search predicate introduced by QLever
  FILTER (LANG(?name) = "en") # filter out non-English results
}
TEXTLIMIT 2 # text search keyword introduced by QLever
```
<figure id='fig3'>
<figcaption>Figure 3: Example of a query using prefix search.</figcaption>
</figure>

<style>
/* Work around because otherwise, the syntax highlighter messes up the colouring of the TEXTLIMIT keyword. */
#site-main > div > article > section > div > div:nth-child(26) > pre > code > span:nth-child(24) {
  color: #66d9ef !important;
}
</style>

The results now include variations like "astrophysicist" and the previously retrieved "astrophysics".
But what the new feature does is not only to return results for the prefix search but also to return the completed keywords that were used for the result. Here we can see the first five results we get if we run the query on the Wikidata dataset (note the new column `?ql_matchingword_text_astrophy`):

?ql_matchingword_text_astrophy | ?name |	?text
--- | --- | ---
astrophysicist | John C. Mather |	John Cromwell Mather (born August 7, 1946, Roanoke, Virginia) is an American astrophysicist, cosmologist and Nobel Prize in Physics laureate for his work on the Cosmic Background Explorer Satellite (COBE) with George Smoot.
astrophysicist | John C. Mather |	Mather is a senior astrophysicist at the NASA Goddard Space Flight Center (GSFC) in Maryland and adjunct professor of physics at the University of Maryland College of Computer, Mathematical, and Natural Sciences.
astrophysics | Vitaly Ginzburg |	He also headed the Academic Department of Physics and Astrophysics Problems, which Ginzburg founded at the Moscow Institute of Physics and Technology in 1968.
astrophysicist | Vitaly Ginzburg	| Soviet astrophysicist Vitaly Ginzburg said that ideologically the "Bolshevik communists were not merely atheists, but, according to Lenin's terminology, militant atheists" in excluding religion from the social mainstream, from education and from government.
astrophysicist | Adam Riess | Adam Guy Riess (born December 16, 1969) is an American astrophysicist and Bloomberg Distinguished Professor at Johns Hopkins University and the Space Telescope Science Institute.

But why is this feature useful? First of all, it allows the user to use the completed keywords just like a normal variable. This makes many functionalities possible like adding the completed keywords to the result, filtering, sorting, or grouping by them. Another application could be to use the completed keywords to perform a sort of auto-completion in a search application, suggesting possible keywords the user might be looking for.

## Improving the Text Search Structure
### Old Structure
Before the restructuring, QLever's text search operation had an inflexible structure. With the help of Figure 4, we will take a deeper look at it.

<figure id='fig4'>

![Old Structure](img/old-structure.png)
<figcaption>Figure 4: Old structure of QLever's text search operation.</figcaption>
</figure>

Figure 4 shows the query tree for the query shown in <a href='#fig2'>Figure 2</a>.
The tree shows how QLever executes a query. The nodes in a query tree correspond to the operations the engine performs. The hierarchy of the tree tells us in which order the operations are executed. Children of a node are executed before the node itself. So the leaves get executed first and the root last. But what does this tell us about the structure of the text search in QLever? We can see that while usually one functionality in the query roughly corresponds to one operation in the tree. This doesn't hold for the text search in QLever. In this old structure, one operation is used to perform all the different functionalities associated with the text search.

But why is this structure bad? There are multiple reasons. First of all, the text search is implemented in one very large block of code, which makes it hard to read and maintain. It also contains a lot of code duplication because the text search operation has specialized methods for functionalities already implemented in a generalized way. And lastly, this structure also makes it hard to implement new features, like the complete word feature. To implement it in this structure we would need to not only change what is read from the text index, but we would also need to adapt nearly all functions of this huge operation, so they can correctly handle and process the new data.

### New Structure
We restructured the text search operation into three smaller, more manageable operations, each handling a specific aspect of the text search functionality. The new operations are as follows:

1. One operation implementing the functionality of `ql:contains-word`
2. One operation implementing the functionality of `ql:contains-entity`
3. One operation implementing the functionality of `TEXTLIMIT`

Figure 5 illustrates the new structure of QLever's text search operations. The new operations are highlighted in blue:
<figure id='fig5'>

![New Structure](img/new-structure.png)
<figcaption>Figure 5: New structure of QLever's text search operations.</figcaption>
</figure>

Figure 5 shows the query tree for the query in <a href='#fig2'>Figure 2</a> with the new structure.
But what is now the advantage of this new structure? Well, first of all, it is easier to understand and read because the code is split up into a few bite-sized pieces, and each only has one functionality. This structure also has less code duplication, because now we make more use of the already implemented operations. Another advantage is that the execution order is now more modular. As you can see in Figure 5, the three new operations are applied all over the tree. This is an advantage because different execution orders of the same query can have different runtimes, so if there is high modularity there are many possible execution orders and much space for optimization. And a big last advantage, that we will also see in action in a second, is that this structure makes the implementation of new features way easier.

### Feature Implementation
With the new structure, adding the prefix search completion feature became straightforward. We only needed to modify the operation implementing the `ql:contains-word` functionality, adding a few lines of code to read additional information from the text index. This modular approach minimized the impact on other parts of the codebase, highlighting the advantages of the new structure.

## Conclusion
The restructuring of QLever's text search functionality has significantly improved the maintainability, readability, and modularity of the code. The new prefix search completion feature exemplifies how these improvements facilitate the implementation of new features. Overall, the enhancements made to QLever demonstrate the importance of a well-structured codebase in supporting ongoing development and feature expansion.


<style>
    figure {
        transform: translateY(-0.5em);
    }
    figcaption {
        text-align: center;
        font-size: 0.75em;
        transform: translateY(-1em);
    }
    figcaption.multi-line {
        text-align: justify;
        padding: 0 5em;
    }
</style>
