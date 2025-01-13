---
title: "Semantic SPARQL Templates for Question Answering over Wikidata"
date: 2023-03-31T12:04:30+01:00
author: "Christina Davril"
authorAvatar: "img/project-wikidata-templates/christina_davril.png"
tags: ["wikidata", "sparql", "templates", "nlp", "natural language processing", "question answering", "knowledge base", "knowledge graph", "kgqa", "kbqa"]
categories: ["project"]
image: "img/project-wikidata-templates/wikidata_tastydata_cropped_white.svg"
draft: false
---

Translating natural language (NL) questions to formal SPARQL queries that can be run over Wikidata to retrieve the desired information is still a very challenging task. This Bachelor's project aims to identify "semantic SPARQL templates", i.e. syntactic SPARQL templates based on elements of the semantic structure of the NL question. When put into practice, these templates should improve the reliable, correct answering of NL questions &ndash;  particularly of those for which the required SPARQL query has a surprisingly complex syntax, e.g., containing subqueries.
<!--more-->

That way, we can also pluck the high-hanging fruit in the Wikidata orchard, and get lemons when we want lemons, and apples, when we want apples.

# Content
1. [Introduction](#introduction)
2. [Defining Complex Questions](#defining-complex-questions)
3. [Databases and Benchmarks](#databases-and-benchmarks)
<br>&emsp; 3.1. [Reasons for choosing the pre-existing datasets](#reasons-for-choosing-the-pre-existing-datasets)
<br>&emsp; 3.2. [Reasons for creating and using the *Wikipedia Lists* dataset](#reasons-for-creating-and-using-the-wikipedia-lists-dataset) 
<br>&emsp; 3.3. [Procedure used to create the *Wikipedia Lists* dataset](#procedure-used-to-create-the-wikipedia-lists-dataset)
4. [Towards Semantic SPARQL Templates](#towards-semantic-sparql-templates)
<br>&emsp; 4.1. [Excluded query types](#excluded-query-types)
<br>&emsp; 4.2. [The general structure of SPARQL queries](#the-general-structure-of-sparql-queries)
<br>&emsp; 4.3. [Semantic classification of syntactic building blocks](#semantic-classification-of-syntactic-building-blocks)
<br>&emsp; 4.4. [A closer analysis of templates containing subqueries](#a-closer-analysis-of-templates-containing-subqueries)
5. [Summary](#summary)


# Introduction

To make using knowledge graphs like Wikidata user-friendly, numerous **KGQA systems** (**K**nowledge **G**raph **Q**uestion **A**nswering) have been developed that translate natural language (NL) questions to a corresponding SPARQL query. 

<figure>
<img id="nodes_black" alt="Wikidata pros and cons" src="/../../img/project-wikidata-templates/wikidata_nodes_in_black_modified_narrower.svg">
<figcaption>
<span style="color:#206040;">Fig. 1</span>: Modified version of <a href="https://upload.wikimedia.org/wikipedia/commons/e/ea/Wikidata_nodes_in_black.svg">this image</a>, attributed to <a href="https://creativecommons.org/licenses/by-sa/3.0">Thepwnco, CC BY-SA 3.0</a>, via Wikimedia Commons.
</figcaption>
</figure>

SPARQL (**S**PARQL **P**rotocol **A**nd **R**DF **Q**uery **L**anguage), the semantic query language used for querying Wikidata, contains syntactic elements that often can be mapped in a straightforward way to the semantic components of the information need of the user, as expressed by the NL question they posed. The information need or intent of the user can be understood as the characterization of the set of results the user expects.

<div class="example">
For example, if a user is querying Wikidata for people who were born in Berlin, this property of the people of being born in Berlin can be ensured by adding the triple <span class="mono">?person p:P19/ps:P19 wd:Q64</span>. Here, <span class="mono">wdt:P19</span> links to the property "place of birth" and <span class="mono">wd:Q64</span> links to the Q-item for the city of Berlin, while <span class="mono">?person</span> is a variable. <br><br>
<span class="mono">p</span>, <span class="mono">ps</span> and <span class="mono">wd</span> are Wikidata-internal prefixes representing certain namespaces, while <span class="mono">P</span> and <span class="mono">Q</span> are Wikidata-internal identifiers which discern <b>(Q-)items</b> from <b>properties</b>. Q-items represent <span style="letter-spacing:0.075em;">things</span> in human knowledge, including topics, concepts, and objects. Properties represent attributes of Q-items, including relations between them. While the term <b>entity</b> is often used as a synonym for (Q-)item, it is actually a <a href="https://www.wikidata.org/wiki/Wikidata:Glossary">term encompassing both items and properties</a>.
Together, prefixes, identifiers and an integer number represent the unique <b>URI</b> (Uniform Resource Identifier) of an entity.<br><br>
A <b>(semantic) triple</b> is then a data structure consisting of the three parts <b>subject</b>, <b>predicate</b> and <b>object</b>. Usually, the subject is a Q-item or a variable for Q-items, the predicate is a property or a variable for properties, and the object is either a Q-item, a variable for a Q-item, or a literal. In the literature, the term "triple" is also used for concrete applications of this data structure, such as <span class="mono">?person p:P19/ps:P19 wd:Q64</span>.<br><br>
A property can also be realized using the <span class="mono">FILTER</span> operator, e.g., to output only people who were born before the year 1990: <span class="mono">FILTER(?birth_year < 1990)</span>.
Here, the filter is applied to a variable containing the birth years of people.
</div> 

However, there are cases where the user's intent or information need – as indicated by the semantic structure of their question – is quite simple, but the syntax of the corresponding SPARQL query is rather complex.

<div class="example">
An example for this is given in <span style="color:#206040;">Figure 2</span> below. <br>
Here, the only – and not very apparent – semantic complexity of the user's question is that they are looking to output the <span style="letter-spacing:0.075em;">current</span> political party of German politicians.  
While this is just an innocent-looking adjective in the question the user entered into the system, this information needs to be extracted by a nested SPARQL query.<br>
This is because Wikidata does not have a property like "current political party", but a property "member of political party" which can have multiple objects. These objects can, as so-called <b>statement nodes</b>, be the subject of other triples containing further information (cf. <span style="color:#206040;">Figure 3</span> below). <br>
For example, for each listed party membership for a given politician, there may be the properties "start time" and "end time", stating (as so-called <b>qualifiers</b>) the dates when the politician joined and left that party.<br>
Only by accessing the start times of these memberships, retrieving the latest (maximum) among them using a <b>subquery</b>, and then using this particular date in the outer query, can this information be retrieved.
</div>

<figure>
<img id="question_query_pair" alt="Example of a question-query pair." src="/../../img/project-wikidata-templates/question_query_example.svg">
<figcaption><span style="color:#206040;">Fig. 2</span>: Example of a question-query pair where the question seems simple but requires a subquery to be answered. As can be seen, the user does not need to put in <span style="letter-spacing:0.075em;">questions</span>, but can also enter imperative sentences or even just some keywords.
</figcaption>
</figure>

<figure>
<img id="statement_nodes" alt="Example of statement nodes" src="/../../img/project-wikidata-templates/statement_nodes_lafontaine.svg">
<figcaption>
<span style="color:#206040;">Fig. 3</span>: Example of a statement group containing the first two statements for the property "member of political party" for German politician Oskar Lafontaine, including qualifier information for both values. <a href="https://upload.wikimedia.org/wikipedia/commons/a/ae/Datamodel_in_Wikidata.svg"> This image</a> gives a general overview over the data model used by Wikidata.
</figcaption>
</figure>

Since the way Wikidata stores information is heterogeneous and usually unknown to the KGQA system, it may fail to output the correct results when relying on the information being stored in simple ways (i.e., accessible using a single triple).  
As one possible strategy to overcome this challenge, this project explores the idea of **semantic SPARQL templates** that can be used to capture the potentially complex way information is stored on Wikidata.  
For that purpose, common semantic patterns occurring in natural language questions will be mapped to corresponding (syntactic) SPARQL query templates. Using these mappings, the user's desired information should be output regardless of how it is stored on Wikidata. <br> 

While a review of the existing research in this field revealed some approaches that used pre-made templates, these templates were mostly covering simple queries. Some approaches also included templates specifically made for common SPARQL operators like <span class="mono">COUNT</span> or <span class="mono">ORDER BY</span>. All of these approaches were rule-based and matched certain <span style="letter-spacing:0.075em;">linguistic</span> patterns in the NL question to certain SPARQL templates, rather than mapping more general semantic aspects to templates. 
Strikingly, none of the approaches included any templates containing subqueries. In general, the topic of queries containing subqueries is barely mentioned in the literature.

# Defining Complex Questions

Many papers on KGQA make a distinction between "simple" and "complex" questions, often defining "simple questions" as those that can be answered with queries containing a single triple. While this definition is clear, it also has its weaknesses.

- The number of triples is not a good indicator of complexity: outputting a person only with their date of birth is arguably not considerably more simple than outputting both their date of birth and their date of death. In the latter case, another triple is added to the query text which is almost identical to the first one, only differing in the property being used.
- There is some inherent structural inconsistency in Wikidata. For example, it can prove very challenging to query historical facts as the same place (e.g., the modern United Kingdom) can have many different Q-items referring to it across many historical epochs. However, just because the items that should be part of the output can be instances of different classes, this does not make the <span style="letter-spacing:0.075em;">syntax</span> of the SPARQL query complex, as one can simply use the <span class="mono">VALUES</span> construct to gather all eligible items.

Rather than trying to define absolute complexity, it is more helpful to consider the **relative complexity** between the NL question's semantic structure and the corresponding SPARQL query's syntactic structure, and to focus particularly on examples where there is a large discrepancy between the two.


# Databases and Benchmarks

In order to identify the semantic SPARQL templates, three benchmarks for evaluating KGQA systems which use Wikidata as the underlying knowledge graph (1.–3. in the table below) were analyzed.
In addition, a very small 4<sup>th</sup> dataset called *Wikipedia Lists* was created in order to limit the potential bias of the human-curated (QALD-10, QALD-9-plus) and semi-automatically generated (LC-QuAD 2.0) datasets, specifically made to evaluate KGQA systems.  

<table>
  <caption style="caption-side:bottom;color:gray;padding-bottom:0.5em;padding-top:0.5em;">Table 1: Overview over datasets</caption>
  <thead>
    <tr>
	<th>Name</th><th>Year</th><th>Download</th><th>Publication</th><th>Examples</th><th>Description</th>
	</tr>
  </thead>
  <tbody>
    <tr>
	  <td><b>QALD-9-plus</b></td>
	  <td>2022</td>
	  <td><a href="https://github.com/KGQA/QALD_10/blob/main/data/qald_9_plus/">GitHub</a></td>
	  <td><a href="https://arxiv.org/pdf/2202.00120">Publication</a></td>
	  <td>412</td>
	  <td>This is the training set for the<br> <a href="https://www.nliwod.org/challenge">10<sup>th</sup> QALD Challenge</a>.</td>
	</tr>
	<tr>
	  <td><b>QALD-10</b></td>
	  <td>2022</td>
	  <td><a href="https://github.com/KGQA/QALD_10/tree/main/data/qald_10">GitHub</a></td>
	  <td>[no publication]</td>
	  <td>393</td>
	  <td>This is the test set for the<br> 10<sup>th</sup> QALD Challenge.</td>
	</tr>
	<tr>
	  <td><b>LC-QuAD 2.0</b></td>
	  <td>2019</td>
	  <td><a href="http://lc-quad.sda.tech/">Website</a></td>
	  <td><a href="http://jens-lehmann.org/files/2019/iswc_lcquad2.pdf">Publication</a></td>
	  <td>30.000</td>
	  <td>Only contains "complex" examples,<br> which require at least 2 triples to be<br> answered.</td>
	</tr>
	<tr>
	  <td><b><i>Wikipedia Lists</i></b></td>
	  <td>2023</td>
	  <td><a href="/../../img/project-wikidata-templates/other_files/wikipedia_lists.json">JSON</a></td>
	  <td>[no publication]</td>
	  <td>11</td>
	  <td>Examples are hand-curated following<br> the procedure described later in this<br> section.</td>
	</tr>
  </tbody>
</table>

## Reasons for choosing the pre-existing datasets

- The **QALD datasets** for the 10th QALD (**Q**uestion **A**nswering over **L**inked **D**ata) challenge were chosen as they are currently the most recent benchmarks in the QALD benchmark series, which provides the most widely used benchmarks for KGQA.  
Since existing systems are evaluated on this data, it is only natural to use this data as one empirical basis for this project.  In the QALD series, QALD-9-plus and QALD-10 are the first datasets that are based on Wikidata.
- To include further examples where the SPARQL query is "complex", **LC-QuAD 2.0** (**L**argescale **C**omplex **Qu**estion **A**nswering **D**ataset) was added as another basis. <br>
While the QALD datasets were checked completely, LC-QuAD 2.0, because of its size, was specifically searched for syntactic elements that are typically found in "complex" SPARQL queries.  
Surprisingly, this large dataset does not contain a large variety of query types. The table below gives a short overview over some of the SPARQL constructs and operators that occur or do not occur in LC-QuAD 2.0. Notably, no grouping and no subqueries are featured.  
Furthermore, after identifying certain semantic purposes with high relative complexity, the LC-QuAD 2.0 file was searched for examples of those, e.g., by querying for common English superlatives and superlative suffixes. This was done in the hope of finding additional examples of queries with high relative complexity, but none were found.

<table>
  <caption style="caption-side:bottom;color:gray;padding-bottom:0.5em;padding-top:0.5em;">Table 2: Overview over constructs in LC-QuAD 2.0</caption>
  <thead>
    <tr>
	<th></th><th>SPARQL constructs (in alphabetical order)</th>
	</tr>
  </thead>
  <tbody>
    <tr>
	  <td><b>Contained in LC-QuAD 2.0</b></td>
	  <td><span class="mono">COUNT, DISTINCT, FILTER, LIMIT, ORDER BY, SELECT, WHERE</span></td>
	</tr>
	<tr>
	  <td><b>Not contained in LC-QuAD 2.0</b></td>
	  <td><span class="mono">AVG, BIND, GROUP BY, GROUP_CONCAT, HAVING, MAX, MIN, MINUS,<br> OFFSET, OPTIONAL, SAMPLE,</span> subqueries<span class="mono">, SUM, UNION, VALUES</span></td>
	</tr>
  </tbody>
</table>

<details>
<summary> Other datasets based on Wikidata</summary>
There are other datasets that are based on Wikidata. However, ... <br>
.. some of them do not include any SPARQL gold queries:

<UL>
<LI><i><b>CSQA/CQA</b></i> &ndash; Complex (Sequential) Question Answering</LI>
<LI><i><b>WebQSP-WD</b></i> &ndash; WebQuestionsSP Wikidata</LI> 
</UL>

.. one only contains "simple" examples of questions answerable by a single triple:

<UL>
<LI><i><b>SQWD</b></i> &ndash; Simple Questions Wikidata</LI>
</UL> 

.. and one suffers from the same lack of variety of (complex) examples as LC-QuAD 2.0 &ndash; not featuring any queries containing grouping, aggregations or subqueries &ndash; despite being even larger (117.970 examples):

<UL>
<LI><i><b>KQA Pro</b></i> &ndash; no spelled-out name available</LI>
</UL>
</details>

## Reasons for creating and using the *Wikipedia Lists* dataset

Using Wikipedia lists as a basis for creating additional examples had the advantage that these lists are independent of Wikidata.  
As such, they might contain information that is not available on Wikidata or vice versa. Even so, the Wikipedia lists served as a valuable **"ground truth"** for evaluating the quality of the SPARQL queries &ndash; assuming that the information in Wikipedia lists, which are often created by experts or enthusiasts of the respective topic, is correct. Moreover, Wikipedia lists are a good indicator of what people are interested in, and might query Wikidata about.  
Especially the recreation of <i>full</i> lists given in tabular form, i.e., capturing every row and column using the SPARQL query, turned out to be a valuable starting point. This is because the information to be stored in the table was selected independently of whether it is simple, hard or even impossible to extract using Wikidata.  
This dataset was thus created in an attempt to alleviate the potential bias that researchers might (subconsciously) introduce when creating examples specifically for querying Wikidata. For example, a researcher might tend to include examples into the benchmark that will be *possible* (or even simple) to answer using Wikidata if Wikidata serves as the (only) basis for creating the benchmark.
Using Wikipedia lists was, thus, a way to <b>reduce the <span style="letter-spacing:0.075em;">potential</span> bias</b> of using pre-made datasets, specifically created for the purpose of evaluating KGQA systems.  

<figure>
<img id="football_list" alt="Example 2 of a Wikipedia list" src="/../../img/project-wikidata-templates/danish_football_transfers.svg">
<figcaption><span style="color:#206040;">Fig. 4</span>: This Wikipedia <a href="https://en.wikipedia.org/wiki/List_of_Danish_football_transfers_summer_2008">List of Danish football transfers summer 2008</a> shows that Wikipedia lists can be very detailed and specific, featuring complete information including references about 142 transfers in total &ndash; likely curated by Danish football enthusiasts or experts.
</figcaption>
</figure>

## Procedure used to create the *Wikipedia Lists* dataset

- At least one **base list** (not containing other lists) was selected from the [Wikipedia List of Lists of Lists](https://en.wikipedia.org/wiki/List_of_lists_of_lists) for each of the ten categories ("General reference", "Culture and the arts", ..., "Miscellaneous"). That way, examples for various areas of interests were created.
- For this (base) list, either a **NL question** was formed that could be answered using the list (in 4 cases), or the **full list was re-created** (in 7 cases). In the latter case, relatively small Wikipedia lists in tabular form were preferred.  
Particularly the method of re-creating full lists led to larger queries, containing many triples and operators to retrieve all the information found in the corresponding table – making up, in part, for the small number of examples. 

The dataset file, <a href="/../../img/project-wikidata-templates/other_files/wikipedia_lists.json" style="background-color:#ff9999;display:block;text-align:center;margin:0.75em 0 0.75em 0;">**wikipedia_lists.json**</a> contains the following information for each example:
<UL>
<LI>dataset-internal ID</LI>
<LI>whether it contains aggregation (boolean)</LI> 
<LI>a link to the Wikipedia list</LI>
<LI>a link to the QLever query</LI>
<LI>the query text</LI>
<LI>a comment about whether the results of the Wikipedia list and QLever match, and the main reasons for discrepancies</LI>
</UL>

[QLever](https://qlever.cs.uni-freiburg.de/wikidata/) is the SPARQL engine used to curate the benchmark. More information about QLever can be found [here](http://ad-publications.informatik.uni-freiburg.de/CIKM_qlever_BB_2017.pdf) and [here](https://ad-publications.cs.uni-freiburg.de/ARXIV_sparql_autocompletion_BKKKS_2021.pdf).
Note that unlike the QALD datasets, this dataset's JSON file does not contain an "answers" section. Such a section would have to include more than just a set of URIs or a literal or boolean and would clutter the file. The reader is invited to run the  queries linked in the JSON file for themselves. <br> 
To make the output human-readable, labels for the result entities are already added by the query text. The full links for the Wikidata prefixes were left out to keep query texts short and concise. They are added automatically by QLever when a prefix is input.
See the section [Syntactic considerations](#syntactic-considerations) for four considerations regarding the exact syntax of the query texts.

<figure>
<img id="wiki_list1" alt="Example 1 of a Wikipedia list" src="/../../img/project-wikidata-templates/largest_cities_capitals.svg">
<figcaption> <span style="color:#206040;">Fig. 5</span>: A snippet of the Wikipedia <a href="https://en.wikipedia.org/wiki/List_of_countries_whose_capital_is_not_their_largest_city">List of countries whose capital is not their largest city</a> showing some example entries.
</figcaption>
</figure>

<figure>
<img id="query_for_wiki_list1" alt="Output of query corresponding to Example 1 of a Wikipedia List" src="/../../img/project-wikidata-templates/cities_query.svg">
<figcaption> <span style="color:#206040;">Fig. 6</span>: QLever output for a query re-creating the above Wikipedia list &ndash; also sorted by the population ratio in descending order. <br> Due to data missing on Wikidata, some countries, like India and Palau, do not appear in the QLever output. There are also discrepancies with regard to the population counts used for some countries (e.g., Tanzania and Burundi). <br>The <a href="https://qlever.cs.uni-freiburg.de/wikidata/0Rd648?exec=true">full output</a> of the QLever query can be seen here.
</figure>


Both the Wikipedia lists and Wikidata can hold false or incomplete information. To improve the gold queries in the <i>Wikipedia Lists</i> dataset, a careful comparison of the information according to the Wikipedia list and the Wikidata output was performed. Discrepancies were analyzed and resolved using other sources of information (such as the non-list Wikipedia pages of occurring Q-items). As stated, the dataset file contains the main results of these analyses.<br>
To make competitions like the QALD challenges fair, and their results meaningful, it is important to create benchmarks with a gold query and gold answer that is as correct and complete as possible. A careful analysis of a subset of the results of examples from QALD-9-plus and QALD-10 showed that the (gold query) results are sometimes wrong or incomplete. At the end of the next section, this is discussed in more detail, but before that, the syntactic standard that was used in the Wikipedia lists dataset &ndash; and in part in the templates described in [Section 4](#semantic-templates-for-syntactic-building-blocks-of-sparql-queries) &ndash; needs to be defined.

# Syntactic Considerations

The exact syntax of the SPARQL gold queries in the *Wikipedia Lists* dataset arises from the following considerations:

- Entities are retrieved as subjects or objects of triples by **accessing the values of statements** using the prefixes <span class="mono">p</span> and <span class="mono">ps</span>. The alternative would be to use properties directly with <span class="mono">wdt</span>. By using <span class="mono">p</span> and <span class="mono">ps</span> instead of <span class="mono">wdt</span>, it is ensured that results are output even if the associated statement is not considered truthy. Besides, it may be necessary to access statement qualifiers to answer a question. In order to access these qualifiers (using <span class="mono">pq</span>), the statement needs to be accessed using <span class="mono">p</span> first anyway. 

- When using an URI as object, the "instance of" property (<span class="mono">P31</span>) is combined with the **"subclass" property (<span class="mono">P279</span>)** in the SPARQL gold query if needed. This is because there is some structural variation on Wikidata that is there **by design**. For example, the entity for the city of Berlin (<span class="mono">Q64</span>) is not an instance of the "city" class (<span class="mono">Q515</span>) since it is &ndash; among others &ndash; an instance of the "big city" (<span class="mono">Q5119</span>) class, which is a subclass of the "city" class.

- In the outermost query (or the <span style="letter-spacing:0.075em;">only</span> query in the non-nested case), <b><span class="mono">SELECT DISTINCT</span></b> is used, as in most practical cases, the user will likely not be interested in receiving duplicate output lines. In the nested case, it can be necessary to consider duplicates that are output by subqueries as intermediate results. Here, it depends on the concrete example whether <span class="mono">DISTINCT</span> should be used.

- The **most simple and most natural** query is always preferred. For example, the same variable (e.g., <span class="mono">?person</span>) is used multiple times in order to link entities across multiple triples. While this seems obvious, the benchmarks' gold SPARQL queries sometimes used variants that can be deemed unnatural such as using two different variables for the same entity (<span class="mono">?person1</span> and <span class="mono">?person2</span> and linking them using <span class="mono">FILTER(?person1 = ?person2)</span>).<br>
Another example would be that in this project, <span class="mono">VALUES</span> is preferred to <span class="mono">UNION</span> or <span class="mono">FILTER</span> as it provides the most elegant way of gathering multiple eligible properties or Q-items.
<!---The dark side of the Force is a pathway to many abilities some consider to be unnatural.-->

<details>
<summary> A note about the consistency and correctness of the QALD datasets </summary>
The QALD datasets were quite <b>inconsistent</b> in all the regards listed above except, possibly, the use of the "subclass of" property. For this property, it is hard to define a reasonable usage. <br>
Notably, almost all examples access properties using <span class="mono">wdt</span>, unless a qualifier needs to be accessed. Very few examples use <span class="mono">p</span> (and <span class="mono">ps</span>) without having to.<br><br>
Besides these inconsistencies, some gold SPARQL queries in QALD-9-plus are clearly <b>wrong</b> in the sense that they do not provide the information requested in the NL question.<div class="gap"></div> 
For example, QALD-9-plus, ID 34, <br><span class="inline-example">Which professional surfers were born in Australia?</span><br> has an integer as (gold) answer, corresponding to the number of Australian-born surfers rather than the set of their URIs. QALD-9-plus, ID 244, has the same problem.<br>The gold query for QALD-9-plus, ID 114, is faulty in the sense that it is taken for granted that Mark Zuckerberg is "the founder of Facebook". In a real system, this information would have to be queried.<br><div class="gap"></div>
Other, smaller problems with these datasets are the occurrences of unused variables, inconsistent capitalization of SPARQL operators, and syntax errors. For example, QALD-9-plus, ID 332, lacks a <span class="mono">WHERE</span> in the SPARQL gold query. <div class="gap"></div>
In a few cases, the English and German versions of the NL questions were semantically different. <br> For example, QALD-10, ID 2, has the English question <br><span class="inline-example">among the characters in the witcher, who has two unmarried partners, Yennefer of Vengerberg and Triss Merigold?</span>.<br> In German, this is <br><span class="inline-example">Wer von den Charakteren aus der Geralt-Saga hat zwei unverheiratete Partner, Yennefer of Vengerberg oder Triss Merigold?</span><br> which translates to <br><span class="inline-example">Who among the characters in The Witcher has two unmarried partners &ndash; Yennefer of Vengerberg or Triss Merigold?</span>.<br> <div class="gap"></div>
Furthermore, it is striking that QALD-9-plus, the training set of the 10<sup>th</sup> QALD challenge, does not contain any <span class="mono">ASK</span> queries or queries including subqueries, while the test set, QALD-10, does. <div class="gap"></div>
In general, QALD-10 has similar problems as QALD-9-plus. For example, QALD-10, ID 310, <br><span class="inline-example">Which NBA teams have won the most seasons?</span><br> seems to have an incomplete set of answers. Since after the championship of 2020, the Boston Celtics and Los Angeles Lakers had both won 17 seasons. However, only the Boston Celtics are a correct answer according to the benchmark.
Unfortunately, the links to the endpoints that were used, provided on the QALD website, are broken. The topic of correctness put aside, this example also has an overly complicated gold query, containing redundant clauses and more "code repetition" than necessary.<br><br>
In addition, the naming of variables in QALD-10 was very inconsistent, ranging from descriptive (<span class="mono">?islandGroup</span>) over obscure (<span class="mono">?c, ?bow</span> as variable for rivers ('body of water')) to generic (<span class="mono">?uri, ?val</span>). This negatively affects the usability of the benchmark. <div class="gap"></div>
In order to make official competitions like the QALD challenges fairer to the contestants, and their official ranking/scores more meaningful, the quality of the provided gold queries (leading to the provided answer sets) could be improved.
</details>


# Towards Semantic SPARQL Templates  

While some of the syntactic considerations listed in the last section are also useful when applied to semantic SPARQL templates (which, as a reminder, consist of SPARQL syntax), there must also be a lot of **abstraction** in order to make the templates as generally applicable and clear as possible. <br>
For example, when deriving the templates, any output formatting that was not explicitly "requested" (i.e., part of the semantic purpose or intent) was left out. This includes for example sorting by sitelinks in descending order as a small QoL improvement for the user, who might be more interested in well-known result entities. <br>
A similar example would be to add a language filter to only show, e.g., the English labels of entities. In practice, this is of course very important but it would clutter the templates unnecessarily.
In a similar vein, the templates contain no shorthand notation like so-called <a href="https://www.w3.org/TR/sparql11-query/#QSynTriples">Predicate-Object Lists and Object Lists</a>. They neither contain <a href="https://www.w3.org/TR/sparql11-query/#propertypaths">alternative, negative or inverse paths</a>. This is to make their structure clear and consistent.

In general, it should be noted that this blogpost does not list any SPARQL templates that are fully ready to be put into code. The actual implementation of these templates (or a selected subset of them), will be the subject of future work. <br> 
Then, challenges like having to deal with the (partially) inconsistent structure of Wikidata will have to be faced, e.g., by using <span class="mono">VALUES</span> clauses to capture multiple eligible properties or Q-items, or by exploring different ways to retrieve the same information. <br>

<div class="example">
For example, there is a "twin" class on Wikidata that is very lightly used. <br> Relying on the "twin" class to retrieve twins only yields <span style="color:#4d4d4d;">400</span> individuals. <br> Querying twins indirectly, by checking if they share a birthday with another person who has the same mother, results in <span style="color:#4d4d4d;">8.870</span> people. <br>
However, complicated and indirect is not always better.
In this scenario &ndash; if instead of using the "mother" class, we use the "child of" property and make sure its object is "female", we get slightly fewer results (<span style="color:#4d4d4d;">8.838</span>).
</div>


## Excluded query types
The following query types are not covered by any of the semantic templates:

1. **ASK queries**: queries used to return a boolean indicating whether a proposition used in a question is true or false

<div class="example">An example would be the question <br><span class="inline-example">Is New York City the capital of the US?</span><br> with the proposition 'New York City is the capital of the US', which would return <b>false</b> for a corresponding SPARQL query executed over Wikidata. <br>
This type of query is excluded in this project because it can be considered as an edge case, expressable by a regular non-ASK query. This query would simply return the actual capital of the United States for the user to interpret.
</div>

2. **"Option queries"**: queries that answer questions containing multiple options for the correct answer

<div class="example">An example would be the question <br><span class="inline-example">Who lives longer &ndash; kangaroos or llamas?</span><br>
While questions similar to this occur in the QALD benchmarks, they require specific syntactic constructs (such as <span class="mono">BIND(IF(...)...)</span>) to be answered elegantly within a single SPARQL query. These constructs are not always supported by SPARQL engines.<br> As with ASK queries, questions of this kind can be answered indirectly, simply by returning all the needed information. In the given example, a query would return the life expectancies of both species.</div>

3. **"COUNT queries"**: While queries returning a count seem quite frequent &ndash; judging from their frequency in QALD-9-plus and QALD-10 &ndash; the aspect of counting the result entities <span style="letter-spacing:0.075em;">in the end</span> is not informative for identifying semantic SPARQL templates. While the processes behind returning different sets of result entities are various and complex, counting them as a last step is simple and straightforward and can be disregarded here. 

4. **Highly specialized queries**: Queries that require multiple knowledge graphs to be answered (so-called "federated queries") are excluded since this project only covers Wikidata. Moreover, queries that require SPARQL constructs other than the standard SPARQL 1.1 constructs are excluded. This includes complex spatial queries requiring GeoSPARQL functions. To keep things simple, queries specifically targeting strings and requiring the <span class="mono">REGEX</span> function are also excluded.

5. **Two-intention queries**: Some NL questions are about attributes of multiple named entities and/or sets of entities. Intuitively, these questions are compound questions, containing of at least two sub-questions. 
<div class="example">Examples for this are the questions <br><span class="inline-example">Who is the richest bridge player and how rich are bridge players on average?</span><br> (targeting a named entity and a set of entities) and <br><span class="inline-example">How many children did Jacques Cousteau and Pierre-Antoine Cousteau each have?</span> (targeting two named entities).<br>However, the question <span class="inline-example">Who is Barack Obama married to and since when?</span> is a "one-intention query" as it only targets a single named entity's attributes.</div>
To keep things simple, only questions targeting one named entity or one set of entities are covered.

Note that, in general, this work does not cover all that is technically possible using SPARQL queries. Instead, it aims to cover the functionality that is needed to answer **typical** NL questions users might pose, as indicated by the benchmarks.


## The general structure of SPARQL queries

Before looking at concrete templates, a rough overview over the structure of SPARQL queries is needed. 

A **graphical overview over list-type queries**, i.e., queries returning a set of output tuples, is given here, and explained in the following:
<a href="/../../img/project-wikidata-templates/list_type_queries.svg" style="background-color:#ff9999;display:block;text-align:center;margin:0.75em 0 0.75em 0;">**list_type_queries.svg**</a>
The graphic covers our whole database but does not go beyond it, in line with the empirical approach outlined briefly in the previous subsection.

**1** The starting point of every SPARQL list-type query is a selection of a set of items of a certain type. Without this, the output would be empty.
Most often, this set of items is built using the properties <span class="mono">P31</span> ("instance of") and <span class="mono">P279</span> ("subclass of") to retrieve the items of one or more Wikidata classes. <br>

<figure>
<img id="1_selection_of_items" alt="Excerpt from the graphical overview, showing step 1" src="/../../img/project-wikidata-templates/1_selection_of_items.svg">
<figcaption> <span style="color:#206040;">Fig. 7. Syntactic SPARQL constructs used to achieve the semantic purpose are given in light grey.</span>
</figure>

<details>
<summary>Queries accessing named entities directly </summary>
There are queries that do not begin with a selection of Q-items belonging to a certain class. Instead, they start by accessing concrete named entities. Apart from this, these queries work just the same as the list-type queries.<br> There can even be a conditional selection (described in <b>2b</b>) of the set of items because the NER/NED (Named Entity Recognition / ... Detection) linker might output multiple candidates.<br> If the user provides additional information about the named entity in their question, this information can be used for disambiguation. For example <br><span class="inline-example">How many children did jane seymour have? Wife of henry viii</span><br> allows to disambiguate the historical person from the British actress as well as from numerous other people with the same name.
</details>

**2a** In some cases, the selection of items is complete at this point. No further conditions are applied to the basic set of items selected in **1**. The output is configured directly (**3**).

**2b** If the set of items should be filtered further, it is often enough to add one or more (potentially interlinked) triples. Of those, the first triple usually has the item in question as subject. <span style="letter-spacing:0.075em;">Abstract</span> properties that can be checked in this way will be called **simple properties**. Most often, they are checked by putting in a certain object (a URI or a literal) at the end of the triple chain. <br> 

<figure>
<img id="2_accessing_simple_property_values" alt="Excerpt from the graphical overview, showing step 2b, 1/3" src="/../../img/project-wikidata-templates/2_accessing_simple_property_values.svg">
<figcaption> <span style="color:#206040;">Fig. 8</span>
</figure>

Sometimes, however, the abstract property for which fulfilment should be checked is more complex. In these cases, there are multiple ways to access values of <span style="letter-spacing:0.075em;">abstract</span> **complex properties**, listed below.<br>
Note that filtering the set of items does not mean that we either include or exclude items completely. There can be multiple tuples in the (current) output which belong to the same base item. It is these tuples <span style="letter-spacing:0.075em;">pertaining to a base item</span> that we filter!

1. One thing that can always be done and is sometimes needed, is to use <span class="mono">BIND</span> to **define a new variable** using literals, URIs or existing variables. The variable definition can also contain SPARQL functions (e.g., <span class="mono">YEAR()</span>) or arithmetic expressions (e.g., <span class="mono">?var1/?var2</span> for numeric variables).

2. Another method to check complex property fulfilment is to apply a <span class="mono">**FILTER**</span> to the values of an attribute. A SPARQL filter is a boolean function which is used to check relations between a variable and a literal, or between variables. Operators like <span class="mono">\<</span> cover a whole range of (e.g., numeric) values.<br>
SPARQL filters often contain functions like <span class="mono">LANG</span> to access the language of the item's label, or <span class="mono">DAY</span>, <span class="mono">MONTH</span> and  <span class="mono">YEAR</span> to access the corresponding part of a date object (a type of literal).

3. One may also need to group by the items if items have multiple tuples in the (current) output, and **filter by an aggregated value** for an attribute (e.g., their <span class="mono">COUNT</span>) to check the fulfilment of a property. In the grouped case, the tuples are filtered using <span class="mono">**HAVING**</span>, which works like <span class="mono">FILTER</span> but is used with groups.

4. To get the **maximum or minimum** value of an attribute across all items or groups of items, it is easiest to <span class="mono">ORDER BY</span> that attribute, and to use <span class="mono">LIMIT 1</span> to get the top value according to this ranking. There are also variations using <span class="mono">LIMIT j</span> and <span class="mono">OFFSET k</span> where <span class="mono">j, k ≥ 1</span>.

<figure>
<img id="2_accessing_complex_property_values_A" alt="Excerpt from the graphical overview, showing step 2b, 2/3" src="/../../img/project-wikidata-templates/2_accessing_complex_property_values_A.svg">
<figcaption> <span style="color:#206040;">Fig. 9</span>
</figure>

5. **Subqueries** can be used to retrieve values (URIs or literals) needed for filtering. <br>
This is, for example, necessary when the complex property contains information stored in a statement qualifier in a statement group with (potentially) multiple statements, and when an aggregated value for the qualifier value is required. Usually, the <span class="mono">MAX</span> or <span class="mono">MIN</span> across statement qualifier values is queried.<br>
To put it more simply, the subquery is needed to figure out the concrete value to be used for the property fulfilment check that is performed in the outer query. This complex topic is explored in more detail in [Section 4.4](#a-closer-analysis-of-templates-containing-subqueries).

<figure>
<img id="2_accessing_complex_property_values_B" alt="Excerpt from the graphical overview, showing step 2b, 3/3" src="/../../img/project-wikidata-templates/2_accessing_complex_property_values_B.svg">
<figcaption><span style="color:#206040;">Fig. 10</span>
</figure>

**3** In some cases, it is sufficient to output the (clickable) URIs of the output set of items. In other cases, one may also want to output certain attributes for them. Optionally, we can sort the output by one or more of these attributes. All of this is done by accessing these attributes &ndash; optionally modifying them &ndash; and putting their respective variables into the <span class="mono">SELECT</span> statement. <br>
Being properties, attributes can be accessed directly by using <span class="mono">wdt</span>, or indirectly using <span class="mono">p</span> and <span class="mono">ps</span>.<br>
If the attribute corresponds to a statement group with more than one value, or if the attribute corresponds to a qualifier of a statement group with multiple values, there will be multiple tuples for the same item in the output.<br> One can then group by the item and aggregate over the attribute's values (with <span style="letter-spacing:0.075em;">one</span> resulting value). <br>
In a nutshell, everything that can be applied for checking property fulfilment (**2b**) can also be applied to configure the output.<br> The only difference is that the set of output items (i.e., the involved Q-items) stays the same. However, the concrete **set of attributes** to be output for the items can be modified. 
<figure>
<img id="3_output_config" alt="Excerpt from the graphical overview, showing step 3" src="/../../img/project-wikidata-templates/3_output_config.svg">
<figcaption><span style="color:#206040;">Fig. 11</span>
</figure>


## Semantic classification of syntactic building blocks

<a href="/../../img/project-wikidata-templates/other_files/semantic_sparql_templates.pdf" style="background-color:#ff9999;display:block;text-align:center;margin:0.75em 0 0.75em 0;">**semantic_sparql_templates.pdf**</a>
gives an overview over the basic syntactic elements used to realize different semantic purposes - the (basic) semantic SPARQL templates. Due to the compositional nature of SPARQL queries, these templates do not abstract from full queries but from their syntactic building blocks, so-called **query elements**. Even the larger templates for queries containing subqueries are such building blocks. In the next section, we will see an example for this.<br>
Once again, note that the templates are abstractions, and that a lot of details have to be considered when implementing them in a real program.


## A closer analysis of templates containing subqueries

Having listed the most important syntactic building blocks, their general semantic purposes, and how they can be combined, the templates with the highest syntactic complexity will be looked at more closely: the ones containing subqueries (Variants 1-6 in the <a href="/../../img/project-wikidata-templates/other_files/semantic_sparql_templates.pdf">PDF</a>).<br>
For that, their semantic purposes are described in more detail. In addition, for each variant, an example from the data is provided to illustrate it.

**Variant 1** seems to be the most common variant. It retrieves aggregated values across groups (e.g., grouping by the items) in the inner query and outputs tuples matching these values in the outer query. <br>
Since tuples' values should match aggregated values, <span class="mono">MAX</span> and <span class="mono">MIN</span> are the only generally meaningful aggregation functions to be used here.<br><br>
From a semantic point of view, queries requiring this variant can be called **superlative queries**, as they involve extreme values for a certain attribute. Linguistically, this is often represented by a (superlative) adjective such as <span class="inline-example">largest</span> or <span class="inline-example">first</span> in the NL questions<br>
In some cases, however, the extreme value of the attribute is not mentioned explicitly in the NL question. For instance, the present tense and lack of other time indicator in the question <span class="inline-example">How many inhabitants does Paris have?</span>, suggests that the user is looking for the most recent population information for the city. There is no need for a superlative or a word like <i>currently</i>.<br><br>
The question-query-pair given initially on this website (<span class="inline-example">German politicians with their current party</span>) is another example of Variant 1; the word <span class="inline-example">current</span> being the "superlative adjective" in the sense that "current" can be understood as "most recent". The example given for Variant 6 below is also an example for Variant 1. There, the yellow boxes mark the inner queries of two Variant 1 subqueries, sharing the query in the light blue box as their outer query.<br><br>
Whether a subquery is needed for **superlative queries**, depends on how the information is stored in Wikidata:
<UL>
<LI> We only need Variant 1 if the "superlative attribute" involves a <b>statement qualifier</b>.</LI>
<LI> If the "superlative attribute" only involves <b>statement values</b>, one can group by the items and aggregate over the property in question.</LI>
<LI> In rare cases, there are Wikidata properties that natively capture extreme values and can just be used directly. Examples of these <b>"extreme properties"</b> are <span class="mono">P610</span>, "highest point", storing the "point with highest elevation in a region, or on the path of a race or route", or <span class="mono">P1872</span>, "minimum number of players" of a game.</LI>
</UL>

**Variant 2** combines an aggregation at group level with an aggregation across all tuples. <br>
An example for this is QALD-10, ID 23, <span class="inline-example">How many spouses do head of states have on average?</span>. In the query, the <span class="mono">COUNT</span>s for distinct spouse statements for each head of state are extracted. Then, the <span class="mono">AVG</span> (average) value of these counts is calculated.<br>

<figure>
<img id="variant_2" alt="Example query for Variant 2" src="/../../img/project-wikidata-templates/heads_of_state.svg">
<figcaption><span style="color:#206040;">Fig. 12</span>: QALD-10, ID 23. The inner query is used to retrieve the number of spouses for each head of state. The outer query averages over that number. Note that the line numbers do not begin with 1 since the lines with prefixes are not depicted. <a href="https://qlever.cs.uni-freiburg.de/wikidata/ltZJR3">QLever Link</a>
</figure>

**Variant 3** contains the same aggregations as Variant 2, but uses the twice-aggregated values to filter the tuples. QALD-10, ID 310, <span class="inline-example">Which NBA teams have won the most seasons?</span>, is an example for this variant. Since the gold query for this example contained redundant structures, unnecessary code repetition, and a wrong answer, an improved version of it was created.<br>

<figure>
<img id="variant_3" alt="Example query for Variant 3" src="/../../img/project-wikidata-templates/nba_winners.svg">
<figcaption><span style="color:#206040;">Fig. 13</span>: QALD-10, ID 310. The inner query is used to retrieve the number of won seasons for each NBA team. The outer query filters out the teams with a non-maximum number of won seasons. <a href="https://qlever.cs.uni-freiburg.de/wikidata/sTsbeL">QLever Link</a>
</figure>

**Variant 4** uses multiple subqueries to each retrieve an aggregated value for a set of items. The values are then combined using <span class="mono">BIND</span>, and output.<br>
Semantically speaking, this variant is used for **comparison queries**, which &ndash; for the purpose of this project &ndash; are queries comparing the same (aggregated) attribute for different sets of items.<br>
An example for this is QALD-10, ID 291, <span class="inline-example">Which archipelago has more islands: the Galápagos Islands or the Hawaiian Islands ?</span>. 
For **comparison queries**, indicating the relation between values, one can also just output the values to be compared and leave their interpretation to the user.

<figure>
<img id="variant_4" alt="Example query for Variant 4" src="/../../img/project-wikidata-templates/galapagos_hawaiian.svg">
<figcaption><span style="color:#206040;">Fig. 14</span>: QALD-10, ID 291. The inner query is used to <span class="mono">COUNT</span> the islands of each archipelago. The outer query uses <span class="mono">BIND(IF...))</span> to define the output variable as the larger count among the two. (<span class="mono">IF</span> is not highlighted in the example because QLever does not support it yet. Therefore, no QLever URL is provided. While this particular example falls into the category of "option query", which was listed among the excluded query types, it is still shown here for lack of alternatives.
</figure>

**Variant 5** retrieves an aggregated attribute value across a set of tuples (e.g., a set of property statement values for a named entity) to use as a reference value. This reference value is then used in the outer query to filter by attribute values that are aggregated across groups.<br>
An example for this is <i>Wikipedia Lists</i>, ID 4, <span class="inline-example">Pandemics that were worse than Covid?</span>. 
From a semantic point of view, this is, again, a **comparison query**. The difference from Variant 4 is that the relation between the values should not just be displayed but is used for filtering.

<figure>
<img id="variant_5" alt="Example query for Variant 5" src="/../../img/project-wikidata-templates/pandemics.svg">
<figcaption><span style="color:#206040;">Fig. 15</span>: <i>Wikipedia Lists</i>, ID 4. The inner query is used to retrieve the highest "number of deaths" value for the Covid-19 pandemic. The outer query is then used to retrieve the highest "number of deaths" for any pandemic, and outputs the pandemic if its highest "number of deaths" value is higher than the one of Covid-19. <a href="https://qlever.cs.uni-freiburg.de/wikidata/AQqXMx">QLever link</a> (query slightly modified to run in the current version of QLever)
</figure>

**Variant 6** filters out everything but the top k (k &#8712; &#8469;) tuples according to an ordering by an attribute, and then aggregates over the attribute's values.<br>
An example for this is given by QALD-10, ID 203, <span class="inline-example">What is the combined total revenue of three largest Big Tech companies ordered by number of employees?</span>.<br>
Since the gold query of this example in the dataset was faulty, a corrected version of it was created. This version contained Variant 1 twice and Variant 6 once, and is shown in the image below.<br>

<figure>
<img id="variant_6" alt="Example query for Variant 6, containing nested subqueries" src="/../../img/project-wikidata-templates/nested_subqueries.svg">
<figcaption><span style="color:#206040;">Fig. 16</span>: QALD-10, ID 203, fixed query. The inner queries of Variant 1 are surrounded by yellow boxes. They are used to retrieve the latest "point in time" for which information about the company's total revenue and about its number of employees is available. <br>
The inner query of Variant 6 is highlighted by a light blue box. In this subquery, everything but the top 3 tuples when ranked by descending number of employees were filtered out. An aggregation over those 3 tuples is then performed by taking the <span class="mono">SUM</span> of their revenue values in the outer query. <a href="https://qlever.cs.uni-freiburg.de/wikidata/XYoBPU">QLever Link</a>
</figure>

# Summary

This project explored the topic of semantic SPARQL templates and identified a set of concrete templates that can be used in practice to improve the performance of KGQA systems &ndash; particularly on examples with high relative complexity. <br>
The project also highlighted that there is still much room for improving the KGQA benchmarks, both with regard to the variety of their examples, and with regard to the correctness of their gold queries and answers. <br>
The idea of using Wikipedia lists as a basis and ground truth for creating KGQA benchmarks was introduced and its merits highlighted. With the creation of the <i>Wikipedia Lists</i> dataset published here, it was also applied in practice. <br>
Future work should tackle a clearer, formal description of the templates that have been identified so far. This includes a clearer description of how the templates can be nested and combined with each other than what was presented here. To do so, semantic and syntactic aspects must fully be separated, and the smallest, truly <i>atomic</i> semantic building blocks must be identified in a more fine-grained analysis and mapped to their possible syntactic realizations. <br>
All the while, the usefulness and completeness of the templates collected so far needs to be assessed in the light of existing and new data, and adjustments need to be made accordingly. The resulting formal concretization may then serve as the basis of a semantic-templates-based KGQA system. <br>
Research in this area could also benefit greatly from improving and extending the currently available empirical basis by creating better benchmarks or building onto existing ones. <br> 
Ideally, the resulting benchmarks would then capture the full spectrum of different syntactic structures needed to adequately answer users' questions, including various types of queries containing subqueries. <br>
Moreover, these benchmarks would provide the best possible gold query and answer set (with the highest possible F-measures) for each question, as characterized by other sources of information (e.g., Wikipedia lists).

<style>
.example {
  background-color: #c3e0ee;
  color: black;
  border: 5px #99ddff solid;
  border-radius: 10px;
  padding: 1.5em;
  margin: 0 0 1.75em 0;
  line-height: 1.75;
}

.inline-example {
	font-style: italic;
	color: #19404d;
	text-shadow: 0.5px 0.25px;
}

.mono {
  font-family: monospace;
  color: #4d4d4d;
}

.gap { 
	width:100%; 
	height:15px; 
}

LI { 
	margin-left: 40px;
	color: #4d4d4d;
}

figure {
  margin-top: 0em;
}

img {
  border: 5px gray solid;
  border-radius: 10px;
}

figcaption {
  text-align: center;
  padding: 5px;
  margin-top: -1.4em;
  margin-bottom: 2em;
  max-width: inherit;
  font-size: 75%;
  line-height: 1.5;
}

h1 {
  padding-bottom: 0.25em;
}

h2 {
  padding-bottom: 0.175em;
}

details {
  background-color: #9fdfbf;
  border: 3px #79d2a6 solid;
  padding: 1.25em;
  padding-bottom: 1em;
  margin: 0.25em 0.5em 1.75em 0.5em;
}

summary {
  padding-bottom: 20px;
}

table {
  border: 4px #990000 solid;
  border-radius: 10px;
  border-collapse: collapse;
  line-height: 1.5;
}
</style>

