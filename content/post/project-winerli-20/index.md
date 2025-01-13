---
title: "Introducing WiNERLi 2.0, an extension of WiNERLi"
date: 2023-04-05T18:10:45+02:00
author: "Johanna Götz"
authorAvatar: "img/ada.jpg"
tags: [entity recognition, entity linking, entity categorization, winerli]
categories: ["project"]
image: "img/project-winerli-20/titleimg.jpg"
draft: false
---

*Named-entity recognition* is a task from the area of natural language processing where the goal is to detect *named entities* in text[^wikiner]. *Named entities* are real-world objects ranging from people (e.g. *Elizabeth II*) to places (e.g. *Freiburg*) to events (e.g. *French Revolution*) to more abstract concepts that can be denoted by a proper name[^wikine]. *Named-entity linking*, also known as *named-entity recognition and disambiguation* is the task of assigning a unique identifier to entities mentioned in text[^wikinel]. For example in the sentence *In 1984, Apple launched the Macintosh, the first personal computer to be sold without a programming language.* the word *Apple* refers to the company *Apple Inc.*, not the fruit named *apple*.

This project is about named-entity linking on Wikipedia, so the goal is to detect and assign possible entities in the text of Wikipedia pages. My code and its functionality are based on the work that another student did for their bachelor's project and thesis. This previous system was named *WiNERLi* (short for *Wikipedia Named-Entity Recognition Linking*). I reimplemented and extended most of the code in an attempt to deliver a higher quality result in a more efficient manner. This new version is called *WiNERLi 2.0*. In the following, I will describe the functionality of WiNERLi 2.0, the differences to WiNERLi and present the evaluation results.

<p><b>CONTENTS</b></p>
<ol>
    <li><a href="#aliasmap_gen">Aliasmap generation</a>
        <ol>
            <li><a href="#aliasmap_goal">Approach and goal</a></li>
            <li><a href="#aliasmap_tech">Technical aspects</a></li>
        </ol>
    </li>
    <li><a href="#ner_disamb">Named-entity recognition and disambiguation</a>
        <ol>
            <li><a href="#ner_preproc">Preprocessing</a></li>
            <li><a href="#ner_er_li">Entity recognition and linking procedure</a></li>
            <li><a href="#ner_add_appr">Additional approaches</a></li>
            <li><a href="#ner_eval">Evaluation</a></li>
            <li><a href="#ner_issues">General issues with the evaluation</a></li>
        </ol>
    </li>
</ol>

--------------------------------------------------------------------------------

## Aliasmap generation <a id="aliasmap_gen"></a>

The first step in both this and the previous implementation is the generation of the *aliasmap*.

### Approach and goal <a id="aliasmap_goal"></a>

The *aliasmap* is a database that contains synonym data. Two terms would be considered synonymous if they belong to the same entity. This is achieved by parsing all Wikipedia pages and extracting the information given by the links and their corresponding link texts. The goal is to assign these possible synonyms to the entities, so that later on when a certain term appears in a text, it can be looked up which entities it could possibly belong to.

The previous implementation consisted used the following method to generate the aliasmap:

Every article page constitutes an entity with the page title as the name for the entity.
Every article page in the English Wikipedia was parsed and the links were extracted. Each link consists of the link's display text and its link target. If the link target is an article page, the link text would be assumed to be a synonym of the entity that is defined by the link target's Wikipedia page. For the link text a canonical form named LNRM which removes diacritics and some other characters will be used to achieve more forgiving matching. The resulting pair of data will then be added to a database.

Additionally, the previous implementation used the data given in Wikipedia's infoboxes. Infoboxes consist of key-value pairs. Some of these pairs contain additional information about the name of something or someone, for example for people there exist `native_name`, `birth_name` etc. and for books `title_orig` and `working_title`. There exist a myriad of possible keys in all sorts of infoboxes, so a few ones that clearly give an (alternative) name and thus synonym data were manually selected.

The extracted synonym data will then be processed. For this it is counted how many times a given text (e. g. *"Fear of the Dark"*) links to a certain entity (e. g. Iron Maiden's song *"Fear of the Dark"*, the 2003 horror film *"Fear of the Dark"* etc.). With this information the *relevance score* can be calculated, which is how high the chance is that a certain text links to a certain entity. For example if the text *"Fear of the Dark"* occurs 30 times and in 6 of these cases the entity is the homonymous Iron Maiden song, then the relevance score would be 6/30 = 0.2.
The goal is to be able to find out which entity a certain text most likely belongs to in order to use the data in tasks such as named-entity linking.

### Technical aspects <a id="aliasmap_tech"></a>

Since Wikipedia is written by volunteers, irregularities in the way data is given can occur. Also, many templates, including infoboxes, can have a plethora of aliases which makes it hard to correctly gather all information that is presented in the Wikipedia pages. In the following some differences in the way the data is handled by the previous implementation and by my program will be described. None of the lists in the following detail descriptions are exhaustive by any means.

**XML parser:**
The Wikipedia pages are given in a huge XML file which also contains meta data for each page. This file needs to be parsed in order to be able to process the pages.

Previous implementation: The ElementTree XML parser library was used.

My implementation: The SAX XML parser library is used, however in both implementations the parsing is done iteratively, so it shouldn't make any difference and can be regarded as a personal choice.

**Detection of disambiguation pages:**
Disambiguation pages end with a template like `{{disambiguation}}`. Aliases for this template include `{{disambig}}, {{disamb}}, {{dab}}` and also `{{disambiguation cleanup}}` for diambiguation pages in need of a cleanup.

Previous implementation: Only `{{disambiguation}}` is recognized.

My implementation: All of the templates mentioned are recognized and also some more variations of the `{{disambiguation cleanup}}`, including e. g. `{{disamb cleanup}}`. It is possible that some aliases are still missing but the most common ones will be recognized.

**Link parsing:**
All links are extracted from the Wikipedia articles. The link text and the link target are extracted. The results are filtered to exclude special links, like links to files or images. The details are described in a later section.

Previous implementation: The `wikitextparser` library is used to parse the articles and extract the links and some infoboxes.

My implementation: In some comparisons I made early on I found `wikitextparser` to be fairly slow because the library parses all templates, not only links and infoboxes, so I decided to write my own link parser that parses only links and infoboxes. My parser is written in Cython. The procedure works as follows: Regular expressions are used to jump to the start of links (they start with `[[`) or infoboxes (they mostly start with `{{Infobox`). Then the following text is parsed character by character using a state machine. In this way, the key-value pairs in infoboxes are directly parsed. The template for images inside the article also starts with `[[` and can only be distinguished from normal links by the number of arguments (separated by a pipe character). For these, the caption is parsed because it can also contain links. Then the links and the infobox type and contents are returned.

**Filtering of link targets:**
The link targets might be links to files, images, categories, internal pages etc. but only the ones that correspond to article pages should be used. For example links to images start with `Image:`, for categories it's `Category:` etc.

Previous implementation: Only links starting with `File:, Image:, Category:, wikt:` are recognized.

My implementation: `File:, Image:, wikt:, Wikipedia:, WP:, Media:, Wikisource:, Species:, Commons:, Help:, Talk:` and also interlanguage links (links that link to an article in another language) are recognized. However, due to the nature of Wikipedia this list is not complete either. Category links will be processed differently because these will be ignored for the link parsing but their information will be gathered and exported for later use in named-entity linking.

**"Red links":**
If there's a link whose link target does not exist (yet) it will be displayed in red on Wikipedia articles. Sometimes such links are set because it's expected that the corresponding page will be or should be created, preferably in the near future. These links are template-wise completely normal links that look no different than any other link, however the page they link to and thus the entity they're connected to does not exist.

Previous implementation: Red links were parsed and processed like any other link, leading to entries for entities that don't exist (yet).

My implementation: While processing a Wikipedia article, the page's title is saved in a separate database table, so that it's known which pages really exist. During the final processing and calculation step only all link entries whose link target exists are included in the other database table, so that red links will be excluded since they do not represent valid entities (yet).

**Infobox and synonym keys:**
Infoboxes are parsed in order to extract the infoboxes' categories to determine categories for the entity the Wikipedia page belongs to. In addition, a certain number of attributes are extracted from the infoboxes that are considered to be alternative names for the entity. In infoboxes, information is given in key-value pairs. There exists a variety of keys that can be used to give the name or an alias.

Previous implementation: The infoboxes are also extracted using the results from `wikitextparser`. After this, the parsed templates are filtered. Only the following infoboxes are recognized: `Infobox book, Infobox film, Infobox musical artist, Infobox song, Infobox television, Infobox person, Infobox royalty, Infobox sportsperson, Infobox company". In these the following keys will be recognized as alias information: "name, title_orig, working_title, isbn, film_name, alias, English_title, show_name, show_name_2, native_name, native_name, birth_name, birthname, full_name, fullname" "title, trading_name, romanized_name, ISIN`.

My implementation: All infobox templates whose name starts with "infobox" as well as the `taxobox, mycomorphbox` and `ichnobox` are recognized. For the aliases, all of the keys that were previously used will be recognized as well as `common_name` and `conventional_long_name` (used in country infoboxes). This list is still not complete because a huge number of infoboxes exist and all use a different set of attributes. It's infeasible to try to find all of them, so both approaches use a set of attributes that are contained frequently. Also, the link text is cleaned up in order to remove or replace certain formatting templates that occur frequently, such as `{{&nbsp}}` (space character) or `{{nbhyph}}` (hyphen) to later get a more accurate LNRM representation (described later). Because a huge amount of different templates exist, only very common ones were selected for this.

**LNRM representation:**
In order to get a normalized, canonical version of the link text, the LNRM representation is calculated.

Previous implementation: To compute the LNRM representation the string is converted into lowercase and the following diacritics and characters are removed: `!?.,-_ \\(){}[]#\t\n`

My implementation: The LNRM as described in *Stanford-UBC entity linking at TAC-KBP* by Angel X Chang et al. is used. Here, the string is turned into lowercase and diacritics and all non-alphanumeric ASCII range characters (``!"#$%&\'()*+,-./:;<=>?@[\\]^_`{|}~\t\n ``) are removed.

**Database usage:**
The extracted link data and the resulting aliasmap will be saved in a database for easy querying later on.

Previous implementation: An SQLite database was used into which the data for each processed link was "dumped" at first. After parsing has been finished, all of the data will be read again by the program and will subsequently be processed to calculate the relevance scores. This means that the whole calculation, the replacement of redirects, and which link target and LNRM pair occurs how often, and the relevance score calculation will be done outside of the database and will then be written into the database.

My implementation: An SQLite database is used as well but the table into which the extracted link data is written has a combined primary key on LNRM and the link target columns. When the same combination of values is inserted again, the counter will be increased instead of adding the whole data again as a separate record. The calculation of redirects and relevance scores are done directly in the database itself since the necessary calculations can be easily expressed using SQL queries. In the end, temporary tables will be deleted.

**Multiprocessing:**
Since the Wikipedia pages are all independent of each other, they can be processed using several independent processes.

Previous implementation: The previous implementation does not use any form of multprocessing or multithreading.

My implementation: My implementation uses Python's multiprocessing library. One process, called input process here, is used to read chunks of the bzipped Wikipedia dump using the corresponding index file. Each chunk usually consists of 100 pages and is then enclosed in `<mediawiki>` tags, so that SAX can parse them properly. The chunks are fed into the so-called task queue. Several processes take elements from the task queue, parse them, extract the data and feed the extracted data into the so-called output queue. One process, called output process here, will take elements from the output queue and write it into the database tables. The sizes of the task queue and the output queue are limited because otherwise the input process would fill up the task queue very quickly, causing massive RAM usage.

--------------------------------------------------------------------------------

## Named-entity recognition and disambiguation <a id="ner_disamb"></a>

For the named-entity recognition and disambiguation, pages from the English Wikipedia will be processed in order to detect and assign entities appearing the text. First, the text is preprocessed to clean it up by filtering out unnecessary elements like templates (e.g. for formatting). After this, the cleaned up text will be examined to perform the actual entity linking. For this, the basic procedure of the previous implementation got extended by different experimental approaches that seem plausible to humans. The results were evaluated against a data set manually annotated by humans.

### Preprocessing <a id="ner_preproc"></a>

**Template filtering and clean-up:**
The final result should consist of only plain text that can be fed into the entity recognition procedure.

Previous implementation: All templates including links were completely removed with all of their content.

My implementation: I decided to try to keep content inside of templates where it makes sense. For example tables contain lots of information that might not be running text but still relevant for example in the context of searching, so my goal was to keep the content for entity recognition. Again, there's an enormous number of templates, so it was infeasible to try to process all of them. Because of this, I concentrated on templates that usually contain a decent amount of information or that occur frequently, such as tables, text formatting templates, listings etc. A slight disadvantage is that some data might be kept that is not visible on the page or relevant to it (like the size information for images in infoboxes) because the wiki templates are very flexible in regards to the arguments they take and their order, so it's a very complex task to filter out only those arguments that contain actual content.

### Entity recognition and linking procedure <a id="ner_er_li"></a>

In order to examine the text, it needs to be *tokenized* first. This means that the text will be split up into separate tokens, like words, punctuation marks etc. For example the text *"Elizabeth was born in Mayfair, London, as the first child of the Duke and Duchess of York (later King George VI and Queen Elizabeth The Queen Mother)."* would be split into the following list of tokens: `['Elizabeth', 'was', 'born', 'in', 'Mayfair', ',', 'London', ',' ,'as', 'the', 'first', 'child', 'of', 'the', 'Duke', 'and', 'Duchess', 'of', 'York', '.']` An entity consists of at least one token but there is no upper limit to how many tokens it can consist of. This means that there's no use in examining only single tokens and it is required to generate sequences of several tokens and extend them if necessary. Such sequences of tokens will be called *subsequence* in the following.

The tokenization including part-of-speech tagging was done using the *spaCy* library. SpaCy is a natural language processing library that contains trained machine learning models for a variety of natural language processing tasks. It also contains models for named-entity recognition, though these are not specific to Wikipedia. However, they will be used later on for comparison in the evaluation.

To create a subsequence of tokens which could potentially be an entity, the current token (if the text has just started this will be the first one) of the text will be examined. If the current token belongs to a link, it will be assigned the entity that belongs to the link target and no further steps will be taken for this token. If the token does not belong to a link, a subsequence will be started. To be considered an entity candidate, the following criteria must hold for the subsequence:

1. If the subsequence starts with a puncuation symbol, a new subsequence that starts with the next token will be created.
2. If the subsequence ends in either an adposition or a punctuation symbol, the next token will be added to the subsequence because it's likely that more information related to the current subsequence will follow.
3. If the subsequence's previous token was a definite or an indefinite article, it will be assumed that, together with the current token, the subsequence might be a reference to a category of an entity. The infobox-based category information will be queried and if there's a result, the corresponding entity will be assigned and a new subsequence will be started. (For example if the entity *Queen Mary 2*, which is a ship, is mentioned, it can be assumed that the subsequence with the text *"the ship"* references the entity *Queen Mary 2*.)
4. The subsequence can only end with a token that belongs to certain types of words. In the case here, a subsequence has to end in a noun, a proper noun, a pronoun or additionally an adjective or a number to be considered an entity candidate. If the subsequence ends in another type of word, a new subsequence will be started.
5. If the subsequence starts with a pronoun, it will be checked which entity of the corresponding gender was mentioned last. If such an entity exists, it will be assigned to the pronoun and a new subsequence will be started.
6. For entities that are considered names, all tokens of the subsequence that made up this name were assigned an entity (see the 9. step). This will be called partial entity data. If the current subsequence starts with a token that is part of the partial entity data, the corresponding entity will be assigned to it.
7. If nothing happened which prompted the start of a new subsequence, the current subsequence will be considered an entity candidate. The aliasmap will be queried with the LNRM representation of the subsequence. If there's no result a new subsequence will be started except if the first token of the subsequence is an adjective or an adposition.
8. If the first token is an adjective or an adposition, the first token will be ignored and a new subsequence will start at the following token, effectively trying again without the adjective or adposition. (This is done in case an adjective or an adposition is not relevant to the rest of the subsquence.)
9. If there's a result from the aliasmap query, all the returned data will be examined further. This includes the approaches described later on. After the scores for each of the possible entites has been calculated, the entity with the highest score will be selected. If the score is smaller than the threshold, it will be rejected and a new subsequence is started. If the score is higher, the entity's infobox categories and pronoun will be looked up and the data will be stored for the use with later subsequences. If a pronoun exists, the entity will be assumed to be a person, so the tokens that the subsequence consists of will be stored (e.g. for *"Konrad Zuse"* both *"Konrad"* as well as *"Zuse"* are considered partial entity data and for both of these the entity *Konrad Zuse* will be saved, so that if for example the text *"Zuse"* will be encountered again, it can be traced back to the entity *Konrad Zuse*). The resulting data can be used for the lookups described in the 6. step. Finally, for all the positions of the tokens in the current subsequence, it will be checked if there already exist entities assigned in previous steps. If there aren't, then the current entity will be assigned. If there are, then the previous data is overwritten if the current entity's subsequence is longer (= more specific) or if the length (= specificity) is the same but the current entity's score is higher.

I made a few design decisions that deviate from those employed in the previous implementation:

1. In the previous implementation, to be considered an entity candidate, the subsequence has to end in a (proper) noun or a pronoun. I kept this idea but added adjectives because adjectives, per definition, are attributes to nouns, so they could help finding more specific entities, especially if they're kept as leading adjectives. I also decided to keep numbers (SpaCy can detect both numerical numbers as well as number words) because they are frequently used to give further information as well as dates.
2. So far, subsequences that end in adpositions would be extended which is a feature that I kept, however, I changed the further handling of subsequences that start with adpositions. If for a sequence starting with an adposition no entity can be found in the aliasmap, it won't continue with the next word but it will ignore the adposition and start a new subsequence with the word right after it. This choice was made because adpositions that connect words like nouns together help to add information (example: *Murder on the Orient Express*) while adpositions at the start of a subsequence are not likely to add useful information (example *In December 1989*), however excluding adpositions completely from the start of subsequences did not seem reasonable to me either. It's necessary to note that such decisions, both the ones made in the previous implementation as well as the ones I made, are rather arbitrary as language, grammar and the occurring entity names are quite varied.

### Additional approaches <a id="ner_add_appr"></a>

The basic implementation described above was extended by adding 4 additional approaches that attempt to boost the accuracy of the entity linking task. These approaches are based on common sense and on what might seem plausible to humans but they are experimental in nature.

First of all, instead of filtering out all templates (including wikilinks) which was done in the previous implementation, I keep wikilinks because if there's a link with a certain link text and a certain link target, I can treat the link target as an explicitly given entity that belongs to this text. Basically, links present trivially given entity data that would otherwise be lost if the link data were filtered out. Since the aliasmap is built by using precisely this link information, it'd not be useful to get rid of this information, although it'd make the filtering of the text to get rid of templates, formatting etc. easier. Several of my additional approaches rely on this type of wikilink information. In the following, I will introduce these approaches.

1. Approach 1 is the basic approach that has been used in almost the same way in the previous implementation. For a given sub-sequence, assign the possible entity with the highest relevance score. This score is first of all the basic relevance score given in the aliasmap but it can be modified multiplicatively via user-defined factors by the other approaches if they are used.
2. The idea for approach 2 is that normally a text for which a corresponding Wikipedia page exists only links to this page at its first occurrence. For example if the text *"Queen of England"* occurs several times in a page and always refers to Queen Elizabeth II, then there will normally only be a link on the first occurrence. Based on this I make the assumption that if the text occurs again after it has occurred as an explicit link, it's more likely to refer to the link target of the explicitly given link than to any other possible entity. This is done by checking if the exact sub-sequence had already appeared as a link text in an explicitly given link before and if it has, then the relevance score of the corresponding entity will be multiplied by a user-defined multiplier to boost its relevance.
3. Approach 3 is similar to approach 2 but relies on the link target and the wikilink of possible entities instead of the link text. The idea is again that normally a Wikipedia page is only linked to on the first occurrence of a corresponding text. This approach, however, is about taking all possible entities for the current sub-sequence and checking which of these entities has already appeared as a link target before. For example if there's a link with the text *"Queen of England"* and a wikilink to the article about Elizabeth II, and later the text *"Queen Elizabeth"* occurs for which Elizabeth II is a possible entity, then the entity for Elizabeth II will be considered more likely and the relevance will be multiplied by a user-defined multiplier.
4. Approach 4 assumes that entities might exist in thematic "bubbles", so that for example an article about computers more likely links to other articles about computer-related topics than to completely unrelated topics. For example when the article about Microsoft contains the word *"apple"* it's assumed that it more likely refers to the company Apple Inc. than the fruit because they're both computer-related companies. For this approach, the Wikipedia categories the current page belongs to will be compared to the categories of each page that is a possible entity and an overlap of categories will be calculated as the overlap coefficient. The higher the overlap is, the more likely the entity is thematically related. The overlap coefficient will be multiplied by the user-defined multiplier and the result will be used to boost the relevance of the entities.
5. The idea behind approach 5 is that related articles are more likely to link to each other. For this approach a database will be used that contains information about which page links to which other page. This relation is not necessarily symmetrical but for this approach I will assume that if article A links to article B, then they are related, so if article B contains a text that has the entity belonging to article A as a possible entity, then this entity will be considered more likely than, for example, an entity C whose article does not link to B. For every possible entity that was found for the current sub-sequence, it will be checked if any of them links to the current page and if it does its relevance score will be multiplied by a user-defined multiplier.

Each of the approaches 2-5 can be enabled separately by setting a corresponding multiplier.

Here's an example for the score calculation:

Consider the following text from the article about "*Prince Edward, Earl of Wessex*":

> `Prince Edward was born at 8:20 p.m. on 10 March 1964 at [[Buckingham Palace]], London, as the third son and the fourth and youngest child of [[Queen Elizabeth II]] and [[Prince Philip, Duke of Edinburgh]]. The Queen appointed the Earl of Wessex as [[Lord High Commissioner to the General Assembly of the Church of Scotland]] for 2014.`

In the second sentence, the word *"queen"* is considered an entity candidate and a corresponding entity should be assigned. The aliasmap is queries for *"lnrm__queen"* and the following entity candidates are returned (ordered descendingly by relevance):

| LNRM | Wikilink | Number of occurrences | Relevance |
| ---- | -------- | --------------------- | --------- |
| lnrm__queen | Queen_(band) | 5987 | 0.626714121218465 |
| lnrm__queen | Elizabeth_II | 542 | 0.0567361038417251 |
| lnrm__queen | Queen_regnant | 313 | 0.0327645765728044 |
| lnrm__queen | Queen_(chess) | 253 | 0.0264838270700304 |
| lnrm__queen | Queen_consort | 251 | 0.0262744687532712 |
| lnrm__queen | Queen_(2013_film) | 227 | 0.0237621689521616 |
| lnrm__queen | Monarchy_of_the_United_Kingdom | 127 | 0.013294253114205 |
| lnrm__queen | Queen_(Nicki_Minaj_album) | 124 | 0.0129802156390663 |
| lnrm__queen | Queen_(magazine) | 94 | 0.00983984088767926 |
| lnrm__queen | Queen_(playing_card) | 72 | 0.0075368994033288 |
| lnrm__queen | Queen_Victoria | 67 | 0.00701350361143096 |
| lnrm__queen | Monarchy_of_Canada | 65 | 0.00680414529467183 |
| lnrm__queen | Queen_(Queen_album) | 56 | 0.00586203286925573 |
| lnrm__queen | Queen_(2018_film) | 51 | 0.0053386370773579 |
| lnrm__queen | Queen_Elizabeth_The_Queen_Mother | 48 | 0.0050245996022192 |

If none of the additional approaches is used, the result with the highest relevance would be assigned. This would be *"Queen_(band)"* (relevance: 0.626714121218465) in this example which is clearly not what one would expect in this context.

We assume the following scoring factors for approaches 2-5: `(3.0, 3.0, 3.0, 4.0)`. The initial score for each possible entity is the relevance score from the aliasmap.

For approach 2, it will be checked if the text *"queen"* ever occurred in a link in this article as the link text. This is not the case here.

For approach 3, it will be checked if any of the possible entities ever occurred as the link target on this article. This is the case for *"Elizabeth_II"* because in the text the link "*[[Queen Elizabeth II]]*" appears and *"Queen Elizabeth II"* redirects to *"Elizabeth_II"*. No other possible entity appears in a link target. Thus, *"Elizabeth_II"* whose current score is 0.0567361038417251 will now be 0.0567361038417251 * 3 = 0.1702083115251753.

For approach 4, the overlap between categories of the current article and the possible entities will be calculated. The result are the following overlap coefficients:

| Wikilink | Category overlap coefficient |
| -------- | ---------------------------- |
| Elizabeth_II | 0.14285714285714285 |
| Monarchy_of_Canada | 0.0 |
| Monarchy_of_the_United_Kingdom | 0.0 |
| Queen_(2013_film) | 0.0 |
| Queen_(2018_film) | 0.0 |
| Queen_(Nicki_Minaj_album) | 0.0 |
| Queen_(Queen_album) | 0.0 |
| Queen_(band) | 0.0 |
| Queen_(chess) | 0.0 |
| Queen_(magazine) | 0.0 |
| Queen_(playing_card) | 0.0 |
| Queen_Elizabeth_The_Queen_Mother | 0.10714285714285714 |
| Queen_Victoria | 0.0 |
| Queen_consort | 0.0 |
| Queen_regnant | 0.0 |

The resulting value will then be multiplied with the scoring factor (4.0) minus 1 (4.0 - 1 = 3.0) and 1 will be added, so that the range of possible values will be from 1.0 to 4.0:

| Wikilink | Calculated scoring factor |
| -------- | ------------------------- |
| Elizabeth_II | 1.4285714285714286 |
| Monarchy_of_Canada | 1.0 |
| Monarchy_of_the_United_Kingdom | 1.0 |
| Queen_(2013_film) | 1.0 |
| Queen_(2018_film) | 1.0 |
| Queen_(Nicki_Minaj_album) | 1.0 |
| Queen_(Queen_album) | 1.0 |
| Queen_(band) | 1.0 |
| Queen_(chess) | 1.0 |
| Queen_(magazine) | 1.0 |
| Queen_(playing_card) | 1.0 |
| Queen_Elizabeth_The_Queen_Mother | 1.3214285714285714 |
| Queen_Victoria | 1.0 |
| Queen_consort | 1.0 |
| Queen_regnant | 1.0 |

Thus, *"Elizabeth_II"* will have a new score of 0.1702083115251753 * 1.4285714285714286 = 0.24315473075025043 and *"Queen_Elizabeth_The_Queen_Mother"* will have a new score of 0.0050245996022192 * 1.3214285714285714 = 0.006639649474361086.

For approach 5, it will be checked which of the possible entities link to the current article. This is the case for *"Elizabeth_II"* (current score: 0.24315473075025043), *"Monarchy_of_Canada"* (current score: 0.00680414529467183) and *"Queen_Elizabeth_The_Queen_Mother"* (current score: 0.006639649474361086). The scores will be multiplied by the scoring factor and will become: *"Elizabeth_II"*: 0.7294641922507513; *"Monarchy_of_Canada"*: 0.02041243588401549; *"Queen_Elizabeth_The_Queen_Mother"*: 0.019918948423083258.

The final scores are as follows (sorted descendingly):

| Wikilink | Score |
| -------- | ----- |
| Elizabeth_II | 0.7294641922507513 |
| Queen_(band) | 0.626714121218465 |
| Queen_regnant | 0.0327645765728044 |
| Queen_(chess) | 0.0264838270700304 |
| Queen_consort | 0.0262744687532712 |
| Queen_(2013_film) | 0.0237621689521616 |
| Queen_Elizabeth_The_Queen_Mother | 0.019918948423083258 |
| Monarchy_of_the_United_Kingdom | 0.013294253114205 |
| Queen_(Nicki_Minaj_album) | 0.0129802156390663 |
| Queen_(magazine) | 0.00983984088767926 |
| Queen_(playing_card) | 0.0075368994033288 |
| Queen_Victoria | 0.00701350361143096 |
| Monarchy_of_Canada | 0.00680414529467183 |
| Queen_(Queen_album) | 0.00586203286925573 |
| Queen_(2018_film) | 0.0053386370773579 |

So the text *"queen"* in the given example text will be assigned the entity *"Elizabeth_II"* because it has the highest final score.


### Evaluation <a id="ner_eval"></a>

**Evaluation sets:**

For the evaluation, in the previous implementation two files were used. I used the same evaluation sets to obtain a degree of comparability to at least this implementation.

The first file (later referred to as *Wikipedia w/o links*) was created by hand based on the introduction of 3 different Wikipedia pages (*Konrad Zuse*, *Caesar cipher* and *Binary search algorithm*). In this file, each sentence was split up into tokens and each token got the desired entity assigned (if there is one) and the category the entity should belong to.

The second file (later referred to as *GMB-Walia*) is available at `kaggle.com` [^kagglegmbwalia] and contains a subset of the *Groningen Meaning Bank (GMB)*. This dataset also contains text that is split into tokens and for each token the annotation consists of the part-of-speech tag, the IOB tag and which category the entity belongs to if it is an entity.

**Issues with the evaluation datasets and procedure:**

First of all, I noticed a few issues with the Wikipedia-based evaluation file created for and used in the previous implementation:

1. Several of my additional approaches (2 and 3) rely on explicitly given links as I treat these as trivially given entities. This means that when I use the given file which consists of plain text for the words, information is lost and the approaches can't do anything as they rely on this data.
2. Some words that have an explicitly given link in the Wikipedia page do not have the correspondingly assigned entity in the evaluation file, even though one would assume that this sort of trivial information would be used. There's no explanation given for this.
3. The Wikipedia evaluation set only contains introductory parts of the articles. These parts contain lots of links, so my implementation using explicitly given link information does not have as much entity recognition/linking to do because it will use the link data trivially.
4. The previous implementation evaluates each sentence in the Wikipedia-based evaluation set independently, even if they belong to the same article. This means that the approach to use pronouns and to try and find out which previous entity the pronoun references will not be able to work well unless the entity and a later matching pronoun occur in the same sentence.
5. I had to change the tokenization in the *Wikipedia w/o links* evaluation set because the tokenization of the newer SpaCy version that I used was different from what the file contained. I assume that SpaCy's tokenization has changed in newer versions. In order to be able to compare this data to the results of the newer SpaCy and of my implementation I needed the same tokenization across all evaluation sets. I split up the previous tokens the way the newer SpaCy would do it and assigned the previously assigned entities to all of the single tokens.
6. In the evaluation code from the previous implementation I noticed that for the GMB data set the tokens tagged with the IOB tag `O` (which means that the token does not belong to any chunk) were correctly excluded from the potential entities in the part that determines the quality of the entity categorization, however in the part that evaluates the entity detection these tokens were considered entities which they aren't. This could potentially explain the very low values for recall (but very high precision as all tokens were considered entities) given in the thesis from the previous implementation for the GMB dataset, both for the implementation and for SpaCy.

Because of the 3rd point I have created my own additional evaluation file (later referred to as *Wikipedia w/ links*) that uses longer parts of articles. This file retains all links. I selected a few articles from different topics and prepared them for annotation by using SpaCy to split the text into tokens but reinserting the links again. The annotation was done both by a friend of mine and myself. The links are assigned the entities that correspond with their link target. The rest of the text will be assigned entities based on our own human expectations, so it is of course not completely neutral and should be taken with a grain of salt. For example on the word *"British"* the other annotator assigned the entity *"United Kingdom"* while I intuitively chose *"Great Britain"* which demonstrates that expectations may vary.

For the evaluation of the *Wikipedia w/o links* and the GMB dataset I used an older Wikipedia dump for the creation of the aliasmap. This dump is slightly newer than the one that was used for the evaluation in the previous implementation but still from the same year (2018). This older dump was used to obtain a higher degree of comparability between the previous and my implementation.

For the evaluation of the new Wikipedia dataset (*Wikipedia w/ links*) a current dump was used for the creation of the aliasmap because the pages in the dataset are also current ones.

**Entity detection results:**

(Note: The scoring factors were chosen arbitrarily in an attempt to showcase different possibilities and were not optimized in any way. Approach 5 was evaluated with two different values as its influence depends on the overlap coefficient in addition to the scoring factor, so it will usually be smaller than the static scoring factors used by the other approaches. If multiple approaches resulted in exactly the same values, the lines were combined into one. This happened here because of the small size of the Wikipedia dataset without links and because approaches 2 and 3 rely on link data. This also happened with the GMB-Walia dataset because it contains none of the Wikipedia-specific information that the additional approaches rely on.)

| Dataset             | System      | Threshold | Scoring factors                 | Precision | Recall | F1     |
|---------------------|-------------|-----------|---------------------------------|-----------|--------|--------|
| Wikipedia w/o links | WiNERLi     | 0.5       | N/A                             | 0.5746    | 0.4074 | 0.4768 |
| Wikipedia w/o links | SpaCy (old) | N/A       | N/A                             | 0.5       | 0.1111 | 0.1818 |
| Wikipedia w/o links | SpaCy 3.2.4 | N/A       | N/A                           | 0.587     | 0.1317 | 0.2151 |
| Wikipedia w/o links | WiNERLi 2.0 | 0.5       | `(0, 0, 0, 0)` /<br /> `(1.5, 0, 0, 0)` /<br /> `(0, 1.5, 0, 0)` w/o a/n.         | 0.7628    | 0.5805 | 0.6593 |
| Wikipedia w/o links | WiNERLi 2.0 | 0.5       | `(0, 0, 0, 0)` /<br /> `(1.5, 0, 0, 0)` /<br /> `(0, 1.5, 0, 0)` w/o adj.         | 0.7346    | 0.5805 | 0.6485 |
| Wikipedia w/o links | WiNERLi 2.0 | 0.5       | `(0, 0, 0, 0)` /<br /> `(1.5, 0, 0, 0)` /<br /> `(0, 1.5, 0, 0)` w/o num.         | 0.7309    | 0.7951 | 0.7617 |
| Wikipedia w/o links | WiNERLi 2.0 | 0.5       | `(0, 0, 0, 0)` /<br /> `(1.5, 0, 0, 0)` /<br /> `(0, 1.5, 0, 0)`                  | 0.7118    | 0.7951 | 0.7512 |
| Wikipedia w/o links | WiNERLi 2.0 | 0.5       | `(0, 0, 1.5, 0)` w/o a/n.       | 0.7658    | 0.5902 | 0.6667 |
| Wikipedia w/o links | WiNERLi 2.0 | 0.5       | `(0, 0, 1.5, 0)` w/o adj.       | 0.7378    | 0.5902 | 0.6558 |
| Wikipedia w/o links | WiNERLi 2.0 | 0.5       | `(0, 0, 1.5, 0)` w/o num.       | 0.7333    | 0.8049 | 0.7674 |
| Wikipedia w/o links | WiNERLi 2.0 | 0.5       | `(0, 0, 1.5, 0)`                | 0.7143    | 0.8049 | 0.7569 |
| Wikipedia w/o links | WiNERLi 2.0 | 0.5       | `(0, 0, 0, 1.5)` /<br /> `(1.5, 1.5, 1.5, 1.5)` w/o a/n.       | 0.7688    | 0.6    | 0.674  |
| Wikipedia w/o links | WiNERLi 2.0 | 0.5       | `(0, 0, 0, 1.5)` /<br /> `(1.5, 1.5, 1.5, 1.5)` w/o adj.       | 0.741     | 0.6    | 0.6631 |
| Wikipedia w/o links | WiNERLi 2.0 | 0.5       | `(0, 0, 0, 1.5)` /<br /> `(1.5, 1.5, 1.5, 1.5)` w/o num.       | 0.7357    | 0.8146 | 0.7731 |
| Wikipedia w/o links | WiNERLi 2.0 | 0.5       | `(0, 0, 0, 1.5)` /<br /> `(1.5, 1.5, 1.5, 1.5)`                | 0.7167    | 0.8146 | 0.7626 |
| Wikipedia w/o links | WiNERLi 2.0 | 0.5       | `(0, 0, 0, 3)` /<br /> `(1.5, 1.5, 1.5, 3)` w/o a/n.         | 0.7702    | 0.6049 | 0.6776 |
| Wikipedia w/o links | WiNERLi 2.0 | 0.5       | `(0, 0, 0, 3)` /<br /> `(1.5, 1.5, 1.5, 3)` w/o adj.         | 0.7381    | 0.6049 | 0.6649 |
| Wikipedia w/o links | WiNERLi 2.0 | 0.5       | `(0, 0, 0, 3)` /<br /> `(1.5, 1.5, 1.5, 3)` w/o num.         | 0.7368    | 0.8195 | <span style="font-weight:bold; color:red">0.776</span>  |
| Wikipedia w/o links | WiNERLi 2.0 | 0.5       | `(0, 0, 0, 3)` /<br /> `(1.5, 1.5, 1.5, 3)`                  | 0.7149    | 0.8195 | 0.7636 |
| Wikipedia w/ links  | SpaCy 3.2.4 | N/A       | N/A                           | 0.702     | 0.2213 | 0.3365 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 0, 0, 0)` w/o a/n.         | 0.7691    | 0.6762 | 0.7197 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 0, 0, 0)` w/o adj.         | 0.7481    | 0.7302 | 0.739  |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 0, 0, 0)` w/o num.         | 0.7171    | 0.7169 | 0.717  |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 0, 0, 0)`                  | 0.7028    | 0.7712 | 0.7354 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(1.5, 0, 0, 0)` w/o a/n.       | 0.7701    | 0.68   | 0.7223 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(1.5, 0, 0, 0)` w/o adj.       | 0.749     | 0.734  | 0.7414 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(1.5, 0, 0, 0)` w/o num.       | 0.718     | 0.7209 | 0.7195 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(1.5, 0, 0, 0)`                | 0.7038    | 0.7752 | 0.7377 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 1.5, 0, 0)` w/o a/n.       | 0.77      | 0.6805 | 0.7225 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 1.5, 0, 0)` w/o adj.       | 0.7488    | 0.7345 | 0.7416 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 1.5, 0, 0)` w/o num.       | 0.7181    | 0.7217 | 0.7199 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 1.5, 0, 0)`                | 0.7036    | 0.7759 | 0.738  |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 0, 1.5, 0)` w/o a/n.       | 0.7696    | 0.6787 | 0.7213 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 0, 1.5, 0)` w/o adj.       | 0.7485    | 0.7327 | 0.7405 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 0, 1.5, 0)` w/o num.       | 0.7176    | 0.7194 | 0.7185 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 0, 1.5, 0)`                | 0.7034    | 0.7737 | 0.7368 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 0, 0, 1.5)` w/o a/n.       | 0.7697    | 0.6782 | 0.7211 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 0, 0, 1.5)` w/o adj.       | 0.7486    | 0.7322 | 0.7403 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 0, 0, 1.5)` w/o num.       | 0.7177    | 0.7192 | 0.7184 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 0, 0, 1.5)`                | 0.7034    | 0.7734 | 0.7368 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 0, 0, 3)` w/o a/n.         | 0.77      | 0.6795 | 0.7219 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 0, 0, 3)` w/o adj.         | 0.7494    | 0.7352 | 0.7422 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 0, 0, 3)` w/o num.         | 0.7181    | 0.7204 | 0.7192 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 0, 0, 3)`                  | 0.7043    | 0.7764 | 0.7386 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(1.5, 1.5, 1.5, 1.5)` w/o a/n. | 0.7705    | 0.6822 | 0.7237 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(1.5, 1.5, 1.5, 1.5)` w/o adj. | 0.7492    | 0.7362 | 0.7427 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(1.5, 1.5, 1.5, 1.5)` w/o num. | 0.7186    | 0.7237 | 0.7212 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(1.5, 1.5, 1.5, 1.5)`          | 0.7042    | 0.7779 | 0.7392 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(1.5, 1.5, 1.5, 3)` w/o a/n.   | 0.7708    | 0.6832 | 0.7244 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(1.5, 1.5, 1.5, 3)` w/o adj.   | 0.7499    | 0.739  | <span style="font-weight:bold; color:red">0.7444</span> |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(1.5, 1.5, 1.5, 3)` w/o num.   | 0.7189    | 0.7247 | 0.7218 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(1.5, 1.5, 1.5, 3)`            | 0.7049    | 0.7807 | 0.7409 |
| GMB-Walia           | WiNERLi     |           | N/A                             | 1.0       | 0.2353 | 0.3810 |
| GMB-Walia           | SpaCy (old) |           | N/A                             | 1.0       | 0.0883 | 0.1622 |
| GMB-Walia           | SpaCy 3.2.4 | N/A       | N/A                           | 0.7103    | 0.3968 | <span style="font-weight:bold; color:red">0.5092</span> |
| GMB-Walia           | WiNERLi 2.0 | 0.5       | w/o a/n.                        | 0.3794    | 0.5756 | 0.4574 |
| GMB-Walia           | WiNERLi 2.0 | 0.5       | w/o adj.                        | 0.3747    | 0.5974 | 0.4605 |
| GMB-Walia           | WiNERLi 2.0 | 0.5       | w/o num.                        | 0.344     | 0.6558 | 0.4512 |
| GMB-Walia           | WiNERLi 2.0 | 0.5       | w/ adj./num.                    | 0.3408    | 0.6787 | 0.4538 |

**Entity categorization results:**

| Dataset             | System      | Threshold | Scoring factors                 | Precision | Recall | F1     |
|---------------------|-------------|-----------|---------------------------------|-----------|--------|--------|
| Wikipedia w/o links | WiNERLi     | 0.5       | N/A                             | 0.5588    | 0.1011 | 0.1712 |
| Wikipedia w/o links | SpaCy (old) | N/A       | N/A                             | 0.4717    | 0.1330 | 0.2075 |
| Wikipedia w/o links | SpaCy 3.2.4 | N/A       | N/A                           | 0.4231    | 0.1078 | 0.1719 |
| Wikipedia w/o links | WiNERLi 2.0 | 0.5       | w/o a/n.                        | 0.7778    | 0.1373 | 0.2333 |
| Wikipedia w/o links | WiNERLi 2.0 | 0.5       | w/o adj.                        | 0.7       | 0.1373 | 0.2295 |
| Wikipedia w/o links | WiNERLi 2.0 | 0.5       | w/o num.                        | 0.7662    | 0.2892 | <span style="font-weight:bold; color:red">0.4199</span> |
| Wikipedia w/o links | WiNERLi 2.0 | 0.5       | w/ adj./num.                    | 0.7284    | 0.2892 | 0.414  |
| Wikipedia w/ links  | SpaCy 3.2.4 | N/A       | N/A                           | 0.4674    | 0.1701 | 0.2494 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 0, 0, 0)` w/o a/n.         | 0.7384    | 0.2642 | 0.3891 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 0, 0, 0)` w/o adj.         | 0.7488    | 0.3482 | 0.4754 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 0, 0, 0)` w/o num.         | 0.6736    | 0.2713 | 0.3868 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 0, 0, 0)`                  | 0.6967    | 0.3553 | 0.4706 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 0, 0, 1.5)` w/o a/n.       | 0.7416    | 0.268  | 0.3937 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 0, 0, 1.5)` w/o adj.       | 0.7516    | 0.352  | 0.4795 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 0, 0, 1.5)` w/o num.       | 0.6766    | 0.2745 | 0.3906 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 0, 0, 1.5)`                | 0.6993    | 0.3586 | 0.4741 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 0, 0, 3)` w/o a/n.         | 0.7418    | 0.2682 | 0.394  |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 0, 0, 3)` w/o adj.         | 0.7518    | 0.3523 | 0.4797 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 0, 0, 3)` w/o num.         | 0.6768    | 0.2748 | 0.3909 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 0, 0, 3)`                  | 0.6995    | 0.3588 | 0.4743 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 0, 1.5, 0)` w/o a/n.       | 0.7416    | 0.268  | 0.3937 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 0, 1.5, 0)` w/o adj.       | 0.7512    | 0.352  | 0.4794 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 0, 1.5, 0)` w/o num.       | 0.6766    | 0.2745 | 0.3906 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 0, 1.5, 0)`                | 0.699     | 0.3586 | 0.474  |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 1.5, 0, 0)` w/o a/n.       | 0.7404    | 0.267  | 0.3924 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 1.5, 0, 0)` w/o adj.       | 0.7499    | 0.351  | 0.4782 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 1.5, 0, 0)` w/o num.       | 0.676     | 0.2743 | 0.3902 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 1.5, 0, 0)`                | 0.6981    | 0.3583 | 0.4736 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(1.5, 0, 0, 0)` w/o a/n.       | 0.7404    | 0.267  | 0.3924 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(1.5, 0, 0, 0)` w/o adj.       | 0.7503    | 0.351  | 0.4783 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(1.5, 0, 0, 0)` w/o num.       | 0.6758    | 0.274  | 0.3899 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(1.5, 0, 0, 0)`                | 0.6983    | 0.3581 | 0.4734 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(1.5, 1.5, 1.5, 1.5)` w/o a/n. | 0.7448    | 0.2725 | 0.399  |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(1.5, 1.5, 1.5, 1.5)` w/o adj. | 0.7536    | 0.3565 | <span style="font-weight:bold; color:red">0.4841</span> |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(1.5, 1.5, 1.5, 1.5)` w/o num. | 0.6804    | 0.2793 | 0.3961 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(1.5, 1.5, 1.5, 1.5)`          | 0.7018    | 0.3634 | 0.4788 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(1.5, 1.5, 1.5, 3)` w/o a/n.   | 0.7448    | 0.2725 | 0.399  |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(1.5, 1.5, 1.5, 3)` w/o adj.   | 0.7536    | 0.3565 | 0.4841 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(1.5, 1.5, 1.5, 3)` w/o num.   | 0.6804    | 0.2793 | 0.3961 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(1.5, 1.5, 1.5, 3)`            | 0.7018    | 0.3634 | 0.4788 |
| GMB-Walia           | WiNERLi     | 0.5       | N/A                             | 0.5258    | 0.3115 | 0.3912 |
| GMB-Walia           | SpaCy (old) | N/A       | N/A                             | 0.5001    | 0.5025 | <span style="font-weight:bold; color:red">0.5013</span> |
| GMB-Walia           | SpaCy 3.2.4 | N/A       | N/A                           | 0.4595    | 0.463  | 0.4613 |
| GMB-Walia           | WiNERLi 2.0 | 0.5       | w/o a/n.                        | 0.4571    | 0.3005 | 0.3627 |
| GMB-Walia           | WiNERLi 2.0 | 0.5       | w/o adj.                        | 0.4367    | 0.3009 | 0.3563 |
| GMB-Walia           | WiNERLi 2.0 | 0.5       | w/o num.                        | 0.375     | 0.3001 | 0.3334 |
| GMB-Walia           | WiNERLi 2.0 | 0.5       | w/ adj./num.                    | 0.3625    | 0.3005 | 0.3286 |

**Entity linking results:**

| Dataset             | System      | Threshold | Scoring factors                 | Precision | Recall | F1     |
|---------------------|-------------|-----------|---------------------------------|-----------|--------|--------|
| Wikipedia w/o links | WiNERLi     | 0.5       | N/A                             | 0.4184    | 0.4184 | 0.2857 |
| Wikipedia w/o links | SpaCy (old) | N/A       | N/A                             | N/A       | N/A    | N/A    |
| Wikipedia w/o links | SpaCy 3.2.4 | N/A       | N/A                             | N/A       | N/A    | N/A    |
| Wikipedia w/o links | WiNERLi 2.0 | 0.5       | `(0, 0, 0, 0)` /<br /> `(1.5, 0, 0, 0)` /<br /> `(0, 1.5, 0, 0)` w/o a/n.         | 0.6992    | 0.4195 | 0.5244 |
| Wikipedia w/o links | WiNERLi 2.0 | 0.5       | `(0, 0, 0, 0)` /<br /> `(1.5, 0, 0, 0)` /<br /> `(0, 1.5, 0, 0)` w/o adj.         | 0.6667    | 0.4195 | 0.515  |
| Wikipedia w/o links | WiNERLi 2.0 | 0.5       | `(0, 0, 0, 0)` /<br /> `(1.5, 0, 0, 0)` /<br /> `(0, 1.5, 0, 0)` w/o num.         | 0.6954    | 0.6683 | 0.6816 |
| Wikipedia w/o links | WiNERLi 2.0 | 0.5       | `(0, 0, 0, 0)` /<br /> `(1.5, 0, 0, 0)` /<br /> `(0, 1.5, 0, 0)`                  | 0.6749    | 0.6683 | 0.6716 |
| Wikipedia w/o links | WiNERLi 2.0 | 0.5       | `(0, 0, 1.5, 0)` /<br /> `(0, 0, 0, 1.5)` /<br /> `(1.5, 1.5, 1.5, 1.5)` w/o a/n.       | 0.704     | 0.4293 | 0.5333 |
| Wikipedia w/o links | WiNERLi 2.0 | 0.5       | `(0, 0, 1.5, 0)` /<br /> `(0, 0, 0, 1.5)` /<br /> `(1.5, 1.5, 1.5, 1.5)` w/o adj.       | 0.6718    | 0.4293 | 0.5238 |
| Wikipedia w/o links | WiNERLi 2.0 | 0.5       | `(0, 0, 1.5, 0)` /<br /> `(0, 0, 0, 1.5)` /<br /> `(1.5, 1.5, 1.5, 1.5)` w/o num.       | 0.6985    | 0.678  | 0.6881 |
| Wikipedia w/o links | WiNERLi 2.0 | 0.5       | `(0, 0, 1.5, 0)` /<br /> `(0, 0, 0, 1.5)` /<br /> `(1.5, 1.5, 1.5, 1.5)`                | 0.678     | 0.678  | 0.678  |
| Wikipedia w/o links | WiNERLi 2.0 | 0.5       | `(0, 0, 0, 3)` /<br /> `(1.5, 1.5, 1.5, 3)` w/o a/n.         | 0.7063    | 0.4341 | 0.5378 |
| Wikipedia w/o links | WiNERLi 2.0 | 0.5       | `(0, 0, 0, 3)` /<br /> `(1.5, 1.5, 1.5, 3)` w/o adj.         | 0.6692    | 0.4341 | 0.5266 |
| Wikipedia w/o links | WiNERLi 2.0 | 0.5       | `(0, 0, 0, 3)` /<br /> `(1.5, 1.5, 1.5, 3)` w/o num.         | 0.7       | 0.6829 | <span style="font-weight:bold; color:red">0.6914</span> |
| Wikipedia w/o links | WiNERLi 2.0 | 0.5       | `(0, 0, 0, 3)` /<br /> `(1.5, 1.5, 1.5, 3)`                  | 0.6763    | 0.6829 | 0.6796 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 0, 0, 0)` w/o a/n.         | 0.676     | 0.4239 | 0.5211 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 0, 0, 0)` w/o adj.         | 0.6787    | 0.52   | 0.5888 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 0, 0, 0)` w/o num.         | 0.6132    | 0.4488 | 0.5183 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 0, 0, 0)`                  | 0.624     | 0.5416 | 0.5799 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 0, 0, 1.5)` w/o a/n.       | 0.6792    | 0.4302 | 0.5268 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 0, 0, 1.5)` w/o adj.       | 0.6814    | 0.5265 | 0.594  |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 0, 0, 1.5)` w/o num.       | 0.6165    | 0.4551 | 0.5237 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 0, 0, 1.5)`                | 0.6268    | 0.5482 | 0.5848 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 0, 0, 3)` w/o a/n.         | 0.6794    | 0.4305 | 0.527  |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 0, 0, 3)` w/o adj.         | 0.6818    | 0.5275 | 0.5948 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 0, 0, 3)` w/o num.         | 0.6166    | 0.4554 | 0.5239 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 0, 0, 3)`                  | 0.6272    | 0.5492 | 0.5856 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 0, 1.5, 0)` w/o a/n.       | 0.6791    | 0.4305 | 0.5269 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 0, 1.5, 0)` w/o adj.       | 0.6813    | 0.5268 | 0.5942 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 0, 1.5, 0)` w/o num.       | 0.6164    | 0.4554 | 0.5238 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 0, 1.5, 0)`                | 0.6267    | 0.5484 | 0.585  |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 1.5, 0, 0)` w/o a/n.       | 0.6792    | 0.4307 | 0.5272 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 1.5, 0, 0)` w/o adj.       | 0.6811    | 0.5268 | 0.5941 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 1.5, 0, 0)` w/o num.       | 0.6166    | 0.4561 | 0.5244 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(0, 1.5, 0, 0)`                | 0.6266    | 0.5489 | 0.5852 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(1.5, 0, 0, 0)` w/o a/n.       | 0.6794    | 0.4305 | 0.527  |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(1.5, 0, 0, 0)` w/o adj.       | 0.6814    | 0.5265 | 0.594  |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(1.5, 0, 0, 0)` w/o num.       | 0.6165    | 0.4556 | 0.524  |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(1.5, 0, 0, 0)`                | 0.6267    | 0.5484 | 0.585  |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(1.5, 1.5, 1.5, 1.5)` w/o a/n. | 0.6815    | 0.4353 | 0.5312 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(1.5, 1.5, 1.5, 1.5)` w/o adj. | 0.6826    | 0.5306 | 0.5971 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(1.5, 1.5, 1.5, 1.5)` w/o num. | 0.6189    | 0.4606 | 0.5282 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(1.5, 1.5, 1.5, 1.5)`          | 0.6282    | 0.5527 | 0.588  |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(1.5, 1.5, 1.5, 3)` w/o a/n.   | 0.6815    | 0.4353 | 0.5312 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(1.5, 1.5, 1.5, 3)` w/o adj.   | 0.6833    | 0.5323 | <span style="font-weight:bold; color:red">0.5984</span> |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(1.5, 1.5, 1.5, 3)` w/o num.   | 0.6189    | 0.4606 | 0.5282 |
| Wikipedia w/ links  | WiNERLi 2.0 | 0.5       | `(1.5, 1.5, 1.5, 3)`            | 0.6289    | 0.5544 | 0.5893 |

**Discussion for the results on the Wikipedia-based dataset without links:**

Using leading adjectives or considering adjectives as a part of potential entities while not using any of the other approaches leads to a notable increase (21.5, 15.2 and 24.9 percentage points respectively if numbers aren't used) of the recall for all tasks (detection, categorization and linking) at the cost of a small fraction of precision (below 3 percentage points).

As expected, turning on approaches 2 and 3 (1st and 2nd scoring factors) does not change anything in the context of using the Wikipedia dataset without links since both approaches rely on link data.

Approach 4 (3rd scoring factor) gives a slight increase of the recall in all tasks while in the detection task precision is also slightly higher, however in the categorization and linking task it doesn't have any influence.

Approach 5 (4th scoring factor) gives a marginally better result overall than not using any approach. Interestingly, using the higher scoring value has almost no influence.

Using all approaches at once gives a marginal increase of the recall for entity detection while marginally decreasing precision but overall the difference is tiny compared to using only approach 4 or 5. For the other tasks there's almost no difference between these either.

Conclusion: My implementation, WiNERLi 2.0, performs considerably better than both the previous implementation, WiNERLi, and both the old and the new SpaCy at entity detection on this Wikipedia-based dataset.

**Discussion for the results on the Wikipedia-based dataset with links:**

Using leading adjectives or numbers also leads to a slight increase in recall over using none of these approaches. However, it performs worse than not using adjectives but using numbers in all tasks. The combination of both yields the best recall.

Approach 5 (4th scoring factor) again gives a marginally better result overall but the difference is very small. The same holds for the other approaches as well. The difference between the different values for this approach is insignificant.

Using all approaches combined gives the best scores but the difference is, again, small at around or less than 1 percentage point.

**Discussion for the results on the GMB-based dataset:**

My implementation has a much higher recall than the new SpaCy but a much lower precision at the entity detection task. On the entity categorization task, the precision is almost the same for both but recall is lower for my implementation. This can probably be explained by the higher recall because the categorization comes from infoboxes and if more entities are detected, it becomes more likely that the corresponding Wikipedia articles had infoboxes that had categories outside of what the GMB dataset expects.

When adjectives are used, just like on the Wikipedia dataset, the precision decreased by a few percentage points while the recall increased for entity detection. For entity categorization, the precision decreased by more than 6 percentage points and the precision decreased as well but only marginally. Again, the higher recall at entity detection makes it more likely that more varied categories were encountered.

All additional approaches yield no improvement at all since they're Wikipedia-based. They rely on explicitly given link data, page categories and links from articles to other articles (requiring knowledge about which article the currently processed data belongs to) which does not exist in such a general dataset.

Conclusion: On the GMB dataset the recall was considerably higher than for SpaCy (and the previous implementation, however the reason could be the issue in the evaluation code described above), however at the expense of precision. Both WiNERLi and WiNERLi 2.0 as well as SpaCy don't perform particularly well at entity categorization. This has a lot to do with the categories in use and which categories are expected. The main problem with categorization will be described in more detail later on. It can be considered a weakness of this whole approach that it is highly Wikipedia-specific, so the usefulness for more general tasks is limited.

**Generally, WiNERLi 2.0 outperforms WiNERLi in all tasks, even in the basic case, not using adjectives or numbers. It is hard to say whether this is due to the changes made to the entity recognition/linking procedure or due to a higher quality aliasmap.**

### General issues with the evaluation <a id="ner_issues"></a>

**Entity definition:**
In addition to the issues mentioned above, sometimes it's difficult to determine which entity should be the result, for example if the text *"programmable computer"* appears, should it be considered one entity or should only the *"computer"* part be linked? Should the text *"May 1941"* be treated as one entity or should it be treated as *"May"* and *"1941"*? Should both words be assigned entities or are they considered too general? In the previous implementation, more specific entities were supposed to be used, so the longest possible sequence (using certain rules to determine to which extent a current token could be part of the sequence) of tokens for which an entity can be found would be used. On the other hand, even within Wikipedia year numbers or month names only rarely have links on them. However, the general problem persists: The result of the evaluation depends a lot on the expectations that the creator of the evaluation set had and these can be quite subjective, so it can be hard to determine the objective quality of the entity recognition and entity linking result.

**categorization:**
I did not make any major changes to the way categorization is handled. Just like the previous implementation I use the information from infoboxes and their types, however I parse all infoboxes, so an entity can have several categories assigned to it. In the thesis about the previous implementation it was mentioned that categorization could potentially be improved using the categories Wikipedia articles belong to. I considered this option, however article categories tend to be very specific, for example the article about *Ada Lovelace* has categories like `19th-century British women scientists, 19th-century British writers, 19th-century English mathematicians, Women computer scientists, British countesses` and analogous categories for which a human can infer that the article is about a person but there is no category directly for this. Categories are ordered in hierarchical ways, so for example the category `19th-century British women scientists` itself belongs to the following categories: `19th-century women scientists, British women scientists by century, 19th-century British women, 19th-century British scientists`. All of these again allow a human who knows that women are persons to infer that this category is about persons but, again, it is not explicitly stated anywhere. An example hierarchy would be: `19th-century British women scientists > 19th-century women scientists > 19th-century scientists > 19th-century people by occupation > People by century and occupation > People by century > People by time > People`. It's clearly visible that it takes many steps to arrive at the general `People` category, however even this category again belongs to many other categories (like `Humans, Main topic classifications, Individual apes`), so the question would be: How deeply does one need to traverse the hierarchy to arrive at a suitable and useful result? The `Main topic classifications` category lists very general categories, like `People`, `Business`, `Nature` etc. but the usefulness of these is also questionable or at least limited. On the other hand, the article about *Ada Lovelace* also belongs to the category `Ada (programming language)` whose main category (after several steps up in the hierarchy) belongs to the `Main topic classifications` category `Technology` which is a topic she belongs to but it is not who or what she is.

The general problem here is still which categories specifically are needed for a certain task and also what these categories should be named. If the category named *people* is used for a person then this might mean the same as a category named *person* but for a computer program these are not the same thing. In some cases, category names can be mapped to other equivalent category names but even this is not always possible. For example GMB categorizes the city of London as `geo` while the Wikipedia infobox category is `settlement` which both make sense depending on the exact context. Through several steps in the article categories of Wikipedia London could be categorized into `Main topic classifications` like `Society` but also `Geography`, however this again requires a rather complex category hierarchy system that'd need to be built first.

There have been approaches that train classifiers based on support vector machines using Wikipedia category data to assign a topic to Wikipedia articles [^wikicatsvm]. Approaches such as this seem to be more flexible in regards to which target topics are desired.

--------------------------------------------------------------------------------

At this point I want to thank Axel Lehmann for his enormous help with annotating the dataset so that it would be less biased.

[^wikiner]: https://en.wikipedia.org/wiki/Named-entity_recognition
[^wikine]: https://en.wikipedia.org/wiki/Named_entity
[^wikinel]: https://en.wikipedia.org/wiki/Entity_linking
[^stanfordubc]: https://www.researchgate.net/publication/265107266_Stanford-UBC_entity_linking_at_TAC-KBPis
[^kagglegmbwalia]: https://www.kaggle.com/abhinavwalia95/entity-annotated-corpus
[^wikicatsvm]: https://www.aaai.org/ocs/index.php/AAAI/AAAI17/paper/viewFile/14927/14204
