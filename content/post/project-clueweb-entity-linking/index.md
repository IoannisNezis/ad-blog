---
title: "ClueWeb Entity Linking"
date: 2020-07-30T12:04:17+01:00
author: "Pablo de Andres"
authorAvatar: "img/ada.jpg"
tags: [NER, NED, NLP]
categories: []
image: "img/project-clueweb-entity-linking/workflow.png"
draft: false
---

In this project, Named Entity Recognition and Disambiguation is carried out on the ClueWeb12 dataset.
<!--more-->


<!-- Style for the annotator examples-->
<!-- Used the '!important' tag to override the theme inherited values -->
<style>
  pre { background-color: #E5EFF5 !important; }

  code { color: black !important; }

  code>sub {
      color: purple !important;
      vertical-align: -100% !important;}
  
  code>sup {
      color: teal !important;
      vertical-align: 100% !important;}
  
  .wikiEntity>img {
      max-width: 95%;
      height: auto;
      margin: auto;}
  
  .entity {color: steelblue !important;}
  
  .wikiEntity {
      visibility: hidden;
      position: absolute;
      width: 300px;
      color: navy !important;
      background-color: #eee;
      border-radius: 15px;
      padding: 5px; }
  
  .entity:hover .wikiEntity {visibility: visible;}
</style>

[AD teaching Wiki description](http://ad-wiki.informatik.uni-freiburg.de/teaching/BachelorAndMasterProjectsAndTheses/ClueWebEntityRecognition)

## Content
- [About the project](#about-the-project)

- [Dataset](#dataset)

- [Design](#design)

- [File Execution](#file-execution)

  1. [File Input](#1-file-input)

  1. [Workflow](#2-workflow)

     1. [Text preparation](#2-1-text-preparation)

     1. [NER and NED](#2-2-ner-and-ned)

  1. [File Output](#3-file-output)

     1. [HTML Record files](#3-1-html-record-files)

     1. [Only entities](#3-2-only-entities)

     1. [Statistics file](#3-3-statistics-file)

- [Server execution](#server-execution)

- [Benchmark execution](#benchmark-execution)

  1. [Benchmark Input](#1-benchmark-input)

  1. [Metrics](#2-metrics)

  1. [Benchmark Output](#3-benchmark-output)

- [Final Notes](#final-notes)


## About the project
Given the sentence `<Mount Everest> is <Earth>'s highest mountain`.

We say that `Mount Everest` and `Earth` are Named Entities in that they refer to a real-world object that can be denoted with a proper name. 

The process by which these entities are identified is called Named Entity Recognition (NER).
This can be done by tagging each of the words and tokens with parts of speech (noun, verb, adjective...), known as POS tagging, and choosing a particular tag for entities
(like NNP, or proper noun).

The process that determines that in the example `Earth` refers to the planet, and not to the soil or ground, is called Named Entity Disambiguation (NED).

This Java project seeks to carry out NER and NED in a specific dataset called the [ClueWeb12 dataset](http://lemurproject.org/clueweb12/).
This dataset contains millions of websites in English collected by a web crawler.
The tool used for NER and NED is [Standford's CoreNLP](https://stanfordnlp.github.io/CoreNLP/). It will be presented in the following sections.

A [benchmark](#benchmark-execution) has also been developed, as well as a simple server for [testing NER and NED](#server-execution)


The project is built to be run in a Docker container:

```
sudo docker build -t <name> .
sudo docker run -p <public-port>:<private-port> -it -v <folder-with-input-files>:/input-files <name>
```

An example run with wharfer:

```
wharfer build -t pablo-de-andres .
wharfer run -p 49152:49152 -it -v /nfs/students/pablo-de-andres/input-files:/input-files pablo-de-andres
```

## Dataset
The ClueWeb12 dataset is made up of WARC files.

A WARC (or Web ARChive) is a standard file format for web crawls to aggregate related content and metadata.
A WARC file is a concatenation of WARC records, which consist of a header and a content block.
The header includes information like the type or the length of the record.

The dataset contains 22.447 WARC files taking 389gb compressed. 
These are organised in 4 disks, with 5 segments per disk, up to 20 directories per segment and up to 100 WARC files per directory.
In total there are over 52 million records. Each file takes around 200mb.

The relevant records are HTTP responses that contain the HTML code of a website queried by the crawler.

## Design
The program has been designed to have 3 different execution modes:

- _File execution:_ the main execution. Runs NER and NED on WARC files.

- _Server execution:_ a simple web server to demo NER and NED with an input for text.

- _Benchmark execution:_ to measure the precision and recall between different file executions.

In the following sections we will review each one in detail.

## File Execution
The program receives one or multiple WARC files (a directory), iterates the records inside the compressed files and extracts the cleaned text.
The text is then fed to a Standford CoreNLP pipeline that carries out NER to identify the entities, and NED, by matching them to a Wikipedia URL.

Wikipedia is then queried through their API to extract the desired information which is written in the output files.

The program is run as follows:

```
java -jar dist/NerMain.java <warc-file/folder> [-fineGrained]
```
where `-fineGrained` is a flag that carries out a more detailed NER (slower).
It is set to false by default.

The first step will load Stanford's CoreNLP library, which takes around 1 or 2 minutes.
Then, the program will go through the steps explained in the upcoming sections.

For reference, one execution time for the file `0000wb-00.warc.gz` of 155mb (982mb uncompressed) with 41,356 records is 201 minutes, 
with a total of 814,399 entities detected of which 694,956 are found in Wikipedia.

### 1. File Input
The input for the program should be individual WARC files or directories containing (only) WARC files.
If the input is a folder, the files are iterated and run individually.
### 2. Workflow
#### 2.1. Text preparation
- First, the WARC file records are iterated.
  For efficiency purposes, this is done without decompressing the file fully, with the [Mixnode WARC reader](https://github.com/Mixnode/mixnode-warcreader-java) library. 
  This library allows the program to go through all the records, filtering the HTTP responses to the crawler (response records).

- `Apache HttpComponents` is used to extract the content of the HTTP response. This will be HTML code.

- The relevant content from the HTML is extracted with [boilerpipe](https://code.google.com/archive/p/boilerpipe/).
  This means removing things like tags, images and boilerplate.
  Originally, [Jsoup](https://jsoup.org/) was used to select only the content of the `<p>` tags, but it proved to be too limiting.

- In a previous version (prior to CoreNLP), the text was split into sentences, and the sentences into words with a [BreakIterator](https://docs.oracle.com/javase/8/docs/api/java/text/BreakIterator.html).
  However, CoreNLP has to tokenize the text during its pipeline, so the whole cleaned text can be fed to the library.

#### 2.2. NER and NED
The initial approach was to use Viterbi's algorithm for Hidden Markov Models as presented in the Information Retrieval lecture
([here](https://daphne.informatik.uni-freiburg.de/ws1920/InformationRetrieval/svn/public/slides/lecture-13.pdf) are the slides).
This algorithm uses the probabilities of each word to be a certain POS tag and the probabilities for the transition between tags to find the most probable sequence of tags for a sentence.
Then, the `NNP` (Proper Noun Singular) tag is used to mark entities.

The program was later modified to use Stanford's CoreNLP since it provides more comprehensive methods for NER and NED.
The CoreNLP software allows you to choose which annotators should be applied to the text.
When the pipeline is executed, the annotators are run sequentially (starting on the input text).
The relevant ones for this project are:

 - `tokenize`: Divides the text into tokens (roughly correspond to words).
   <details>
    <summary>Example</summary>

     ```
       Mr. O'Neill thinks that the boys' stories about Chile's capital aren't amusing.
     ```

     becomes

     ```
       Mr.
       O'Neill
       thinks
       that
       the
       boys
       '
       stories
       about
       Chile
       's
       capital
       are
       n't
       amusing
       .
     ```
   </details>

 - `ssplit`: Uses the tokens to split a text into sentences.

 - `pos`: Assigns POS tags to each token per sentence.
   <details>
    <summary>Example</summary>

     ```
       Mr. O'Neill thinks that the boys' stories about Chile's capital aren't amusing.
     ```

     becomes

     <pre style="white-space: pre-wrap;"><code>
     Mr.<sub>NNP</sub> O'Neill<sub>NNP</sub> thinks<sub>VBZ</sub> that<sub>IN</sub> the<sub>DT</sub> boys<sub>NNS</sub> '<sub>POS</sub> stories<sub>NNS</sub> about<sub>IN</sub> Chile <sub>NNP</sub> 's<sub>POS</sub> capital<sub>NN</sub> are<sub>VBP</sub> n't<sub>RB</sub> amusing<sub>JJ</sub> .<sub>.</sub>
     </code></pre>
   </details>

 - `lemma`: Generates the lemmas (base or dictionary form of a word) for each token.
   <details>
    <summary>Example</summary>

     ```
       Mr. O'Neill thinks that the boys' stories about Chile's capital aren't amusing.
     ```

     becomes

      <pre style="white-space: pre-wrap;"><code>
      Mr.<sup>Mr.</sup> O'Neill<sup>O'Neill</sup> thinks<sup>think</sup> that<sup>that</sup> the<sup>the</sup> boys<sup>boy</sup> '<sup>'</sup> stories<sup>story</sup> about<sup>about</sup> Chile <sup>Chile</sup> 's<sup>'s</sup> capital<sup>capital</sup> are<sup>be</sup> n't<sup>not</sup> amusing<sup>amusing</sup> .<sup>.</sup>
      </code></pre>
   </details>

 - `ner`: Recognises the Named Entities. There are 3 different models applied: 3class, 7class, and MISCclass.

    NER can be run with a higher level of detail, at a (rather large) time penalisation.
    As a reference, running the program on `0000wb-32.warc.gz` without applying fine grained detection detects 631,759 wikientities, and takes just over 4 hours.
    Activating fineGrained NER increases the detected wikientities to 782,335; but requires over 14 hours.
    fineGrained can be activated as a flag when running the jar file (see the [execution](#file-execution) section).
   <details>
    <summary>Example</summary>

     ```
       Mr. O'Neill thinks that the boys' stories about Chile's capital aren't amusing.
     ```

     becomes

      <pre style="white-space: pre-wrap;"><code>
      Mr. O'Neill<sub>Person</sub> thinks that the boys' stories about Chile <sub>Country</sub>'s capital aren't amusing.
      </code></pre>
   </details>

 - `entitylink`: Matches the entity mentions to wikipedia entities (over 20 million string matches). 

    For that, `stanford-english-kbp-corenlp-2018-10-05-models.jar` is required. 
    This jar contains a [dictionary-like structure](https://nlp.stanford.edu/pubs/crosswikis.pdf) where strings of entities are mapped to canonical English Wikipedia URLs.
   
    For instance, for the text "FDR" or "Franklin Delano Rooseveltand" it outputs `Franklin_D._Roosevelt`
    that can be appended to `https://en.wikipedia.org/wiki/` in order to obtain the wikiEntity
    (https://en.wikipedia.org/wiki/Franklin_D._Roosevelt).

The full URL is used to connect to Wikipedia's API to extract the name, description, wikidata ID and image URL.
This is done via a query like `https://en.wikipedia.org/w/api.php?action=query&prop=pageprops|pageimages&piprop=original&format=json&titles=Chile`.
A JSON with the information is returned, and used to enrich the entity information.
 <details>
  <summary>Example</summary>

   ```
     Mr. O'Neill thinks that the boys' stories about Chile's capital aren't amusing.
   ```

   becomes

   <pre style="white-space: pre-wrap;">
   <code>Mr. O'Neill thinks that the boys' stories about <span class="entity">Chile <span class="wikiEntity">
   <img src="https://upload.wikimedia.org/wikipedia/commons/7/78/Flag_of_Chile.svg" alt="Image not found">
   <a href="https://en.wikipedia.org/wiki/Chile">Chile</a>
   <i>WikidataId:</i> Q298
   <i>Description:</i> Republic in South America
   </span></span>'s capital aren't amusing.</code></pre>
 </details>

The queried entities are stored within a run for increased efficiency.
By default, a HashMap is used since it is much faster, but using a database instead just requires uncommenting some lines
(view `src/java/core/NamedEntityDisambiguation.java`).


 For more information specific to CoreNLP, [their documentation](https://stanfordnlp.github.io/CoreNLP/index.html) is quite thorough.

### 3. File Output
There are 3 main types of files generated/modified during a run:

1. HTML record files: HTML reconstruction of the text per warc file highlighting the entities found.

2. Only entities: TSV file containing only the entities found.

3. Results file: general summary of the runs (time, number of records, number of entities found...).

#### 3.1. HTML Record files
There is one HTML record file per warc file.

They are located under `output/<timestamp_of_the_run>/html`.

They contain all the extracted sentences in different paragraphs and the entities highlighted in a different colour.
The entities matched to a wikipedia entity (through NED) are marked with an asterisk (`*`),
and the property "data-record" in the paragraph points to the record where the sentence was found. 

 <details>
   <summary>Sample</summary>

   ```html
     <!-- Small excerpt for demo purposes -->
     <!DOCTYPE html>
     <html>
       <head>
         <meta charset="utf-8">
         <meta name="author" content="HtmlWriter for Pablo de Andres' Master Project">
         <meta name="description" content="Identified entities for 0000wb-32.warc.gz">
         <title>0000wb-32.warc.gz</title>
         <link rel="stylesheet" href="../../../css/main.css">
       </head>
       <body>
         <p data-record="clueweb12-0000wb-32-00000"><span class="entity">DAYSTAR*
           <span class="wikiEntity">
             <img src="" alt="Image not found">
             <br><a href="https://en.wikipedia.org/wiki/Daystar_Television_Network">Daystar Television Network</a>
             <br><i>WikidataId:</i>
             <br><i>Description:</i>
           </span>
         </span> NEWSLETTER <span class="entity">NOVEMBER 2009    </span> Once again we want to say '' Thank you '' for your business .</p>
         <p data-record="clueweb12-0000wb-32-00000">For over <span class="entity">30 years*
           <span class="wikiEntity">
             <img src="" alt="Image not found">
             <br><a href="https://en.wikipedia.org/wiki/Porter_five_forces_analysis">Porter five forces analysis</a>
             <br><i>WikidataId:</i>
             <br><i>Description:</i>
           </span>
         </span> we have been providing safety equipment , services , and solutions for many companies in the <span class="entity">Midwest*
           <span class="wikiEntity">
             <img src="https://upload.wikimedia.org/wikipedia/commons/4/4f/Map_of_USA_Midwest.svg" alt="Image not found">
             <br><a href="https://en.wikipedia.org/wiki/Midwestern_United_States">Midwestern United States</a>
             <br><i>WikidataId:</i> Q186545
             <br><i>Description:</i> One of the four census regions of the United States of America
           </span>
         </span> .</p>
       <!-- Small excerpt for demo purposes -->
       </body>
     </html>
  ```

 </details>

#### 3.2. Only entities
 This file contains only the detected entities for each warc file. 

 It has 4 tab separated columns:

  - wikidata ID 

  - WARC record ID

  - Offset (begin)

  - Offset (end)

 They are located under `output/<run_timestamp>/tsv/`.
 
 These files can be used as the input to compute the benchmark statistics (explained in the benchmark execution section). 

 <details>
   <summary>Sample</summary>

   ```
    Q634951 clueweb12-0000wb-00-00000       40      88
    Q84     clueweb12-0000wb-00-00000       92      98
    Q2478   clueweb12-0000wb-00-00000       113     117
    Q21     clueweb12-0000wb-00-00000       148     155
    Q794    clueweb12-0000wb-00-00000       157     161
    Q43     clueweb12-0000wb-00-00000       166     172
    Q2476   clueweb12-0000wb-00-00000       204     208
    Q84     clueweb12-0000wb-00-00000       213     219
    Q664609 clueweb12-0000wb-00-00000       244     253
    Q739700 clueweb12-0000wb-00-00000       296     314
    Q794    clueweb12-0000wb-00-00000       333     337
   ```

 </details>

#### 3.3. Statistics file
  All executions are also logged to a `output/results.log` tsv log file, like the following:

|          date          |    file    |  records | runtime (min) |  wikientities |  total entities |
| -------------------------- | ----------------- | -----: | -------: | -------: | -------: |
| 2020-02-19T01:36:25.695708 | 0000wb-32.warc.gz |  34030 |   270,17 |   631759 |   730767 |
| 2020-02-19T13:52:09.694204 | 0000wb-77.warc.gz |  35179 |   208,35 |   606178 |   713374 |
| 2020-02-20T04:34:22.440702 | 0000wb-27.warc.gz |  38536 |   217,49 |   705700 |   822316 |
| 2020-02-20T17:04:36.226648 | 0000wb-88.warc.gz |  35921 |   168,23 |   722767 |   838604 |
| 2020-02-21T01:25:59.114743 | 0000wb-83.warc.gz |  32003 |   101,20 |   429399 |   496036 |

## Server execution
For demonstration and testing purposes, a simple web server is available.
When started, the server will render a page where the user can input a sample text and see the POS tags and detected entities.

The server execution starts by default at port number 49152, but a different port can be provided.
   ```
     java -jar dist/NerMain.java [-p <port-number>]
   ```
Since the testing will be done in relatively small texts, fineGrained NER is activated.

## Benchmark execution 
The benchmark execution compares the wikidata ids between two different files, grouping them by their WARC record id.

The execution can be done via:
   ```
   java -jar dist/NerMain.java <ground-truth-file> <run-result-file>
   ```

### 1. Benchmark Input
Both files being compared should follow the same format.
Said format is the one presented in the [text record output file](#3-2-only-entities) of the execution,
namely four tab separated columns (wikidata ID, WARC record ID and offsets).

In order to use the [FACC annotated files of the dataset](http://lemurproject.org/clueweb12/FACC1/), 
once the files are downloaded (`wget <path>`) and uncompressed (`tar -zxvf ClueWeb12_*`) some preprocessing is required: 

- From each tsv, only the WARC record ID (first column), the offsets (fourth and fifth coluns) and the freebase ID (eight column) are relevant.

- The freebase IDs are mapped to wikidata IDs.

- The file is formatted to have the wikidata ID in the first column and the WARC record ID in the second. The offset are last

- The new file will be named `<warc-file>.benchmark`.

- A file with all the missing freebase-wikidata mappings is also generated (`<warc-file>.freq_missing_ids`).

The python script to carry all these tasks, as well as the file with the mapping of the IDs is located in `/nfs/students/pablo-de-andres/freebase_wikidata_mapping`.

_IMPORTANT NOTE:_
The offsets from the Freebase annotated version do not match the offsets that this program outputs.
This is due to the fact that CoreNLP receives the cleaned text and computes the offsets from that text.
However, the offsets from Freebase refer to the original WARC file.

### 2. Metrics
- First, the ground truth file is loaded into a helper data structure called `BenchmarkFile`.
It is a `Map`, with the WARC record IDs as keys and for values a nested `Map` with the offsets as keys and the wikidata IDs as values.
(`Map<String, Map<Offset, String>>`).

- Similarly, the file being compared against the ground truth is loaded into another `BenchmarkFile`.

- While loading the second file, four more datastructures are filled:

  - True positives: The entities detected during the run (second file) also present in the ground truth.

  - False positives: The entities detected during the run that are not present in the ground truth.

  - Missing mapping: The entities present in the run and in the ground truth, but missing the mapping from freebase to wikidata ID.

  - Wrong entity: The entities present in both files, but with different wikidata entity IDs assigned.

- The true positives are then used to compute the [_precision_ and _recall_](https://en.wikipedia.org/wiki/Precision_and_recall):
  
  - Precision = true positives / entities detected in the run.

  - Recall = true positives / entities in the ground truth.

### 3. Benchmark Output
Running the benchmark will generate a folder under `output/benchmark/<timestamp>` and dump the four computed data structures
(true positives, false positives missing mapping and wrong entity).
These follow a structure like the other tsv files explained until now, namely four columns, wikidata ID, WARC record ID and offsets.

For the benchmark execution there is also an statistics file located in `output/benchmark/statistics.log`:

|    date    | Benchmark_File | Benchmark_Entities | Identified_File | Identified_Entities | Precision | Recall |
|------------|----------------|-------------------:|----------------:|--------------------:|----------:|-------:|
| 2020-07-22T09:11:43.166127 | 0000wb-00.benchmark | 435401 | 0000wb-00.warc.text_record.tsv | 694536 | 0.35610713684477563 | 0.48089920151530735 |

## Final Notes
This project aims to define the structure and an algorithm for performing NER and NED on the dataset.
Thus, improvements can be made in future iterations:

- Given the size of the dataset, the program has only been run in a very small subset.
  For a full execution, multiple processes, with higher resources are recommended.

- The NER carried out by CoreNLP is limited. Some structures like "Gone with the wind", with POS-tags: \<VBN\> + \<IN\> + \<DT\> + \<NN\> are not detected by default.
  Marking those structures as entities will make structures like "The Zähringen family was \<established in the city\> in 1120. This reduces the accuracy, and could cause shorter, real entities to be skipped when they are part of a bigger structure.
  Nevertheless, this behaviour can be forced by using the `-fineGrained` flag when executing.

- NED is also done via CoreNLP, because the number of disambiguated entities is higher than that of the first algorithm.
  However, the results are not the most accurate, and the usage of another tool could probably improve the results.

  - For example, [AGDISTIS](http://aksw.org/Projects/AGDISTIS.html) is an interesting option.

- The current benchmark implementation does not work to compare to the Freeebase annotated version.
However, it can still be used to compare the results between different versions of the code.
Fixing the benchmark to match the offsets from freebase would require to find the text in the original record file and compute the offset there, which is not a trivial task.
