---
title: "Extension for QLever: Implementing missing SPARQL Expressions"
date: 2024-07-23T15:21:44+02:00
author: "Hannes Baumann"
authorAvatar: "img/ada.jpg"
tags: [QLever, SPARQL 1.1, SPARQL Expressions, C++]
categories: ["project"]
image: "img/project-extension-for-qlever-implementing-missing-sparql-expressions/sparql_expresssions_blog_image.jpg"
draft: true
---

[QLever](https://qlever.cs.uni-freiburg.de/ "QLever") is a database query language engine that follows the [SPARQL 1.1 Query Language standard](https://www.w3.org/TR/sparql11-query/ "SPARQL 1.1 Query Language standard"). It provides an efficient and highly optimized backend for querying RDF ([Resource Description Framework](https://en.wikipedia.org/wiki/Resource_Description_Framework, "Resource Description Framework")) databases based on descriptive triples. Some SPARQL expressions defined by SPARQL 1.1 were still missing, the objective of my project was to implement them.
<!--more-->

# Content
1. <a href="#general-introduction">Introduction QLever</a>
2. <a href="#explanation-relevant-terms">Relevant Terminology</a>

    2.1. <a href="#term-iri-and-uri">IRI and URI</a>

    2.2. <a href="#term-literal">Literals</a>

    2.3. <a href="#term-datetime">xsd:dateTime</a>

    2.4. <a href="#term-daytime-duration">xsd:dayTimeDuration</a>
3. <a href="#implemented-epressions">Implemented Expressions</a>

    3.1 <a href="#datatype-expression">DATATYPE</a>

    3.2 <a href="#strdt-and-strlang-expression">STRDT and STRLANG</a>

    3.3 <a href="#lang-and-langmatches-expression">LANG and LANGMATCHES</a>

    3.4 <a href="#now-expression">NOW</a>

    3.5 <a href="#timezone-and-tz-expression">TIMEZONE and TZ</a>

    3.6 <a href="#struuid-and-uuid-expression">UUID and STRUUID</a>

    3.7 <a href="#hash-expression">Hash Expressions</a>
4. <a href="#conclusion">Conclusion</a>

* * *

# <a id="general-introduction"></a>1. Introduction QLever
QLever ([Github Repository](https://github.com/ad-freiburg/qlever/ "Github Repository")) is an actively developed open project by the Algorithm and Data Structures Chair at the University of Freiburg. It is used in practice by research groups and companies due to its ability to efficiently query relevant datasets like the complete Wikidata, PubChem or UniProt. To put into perspective what a powerful tool QLever actually is, Wikidata is one of the largest actively maintained (hosted by the Wikimedia Foundation) knowledge bases for general information with approximately 19 billion RDF triples. PubChem is the largest freely accessible database for chemical compounds that is maintained by the National Center for Biotechnology Information, and UniProt is a high quality database (approximately 94 billion triples) maintained by relevant european institutions like the European Bioinformatics Institute containing comprehensive knowledge of protein structures and their characteristics.<br>


The engine itself is written in modern C++ and is highly optimized for memory efficiency and query execution speed. Thus, the [hardware requirements](https://github.com/ad-freiburg/qlever/wiki/Using-QLever-for-Wikidata, "hardware requirements") are relatively minimal. The recommended prerequisites for using QLever on the Wikidata dataset are 32 GB RAM with 2TB of Disk Space in combination with a performant CPU. My development notebook, a Lenovo Thinkpad with 16 GB of RAM in combination and an Intel i7 8th Generation, was easily sufficient for running QLever on the the Olympics dataset. This dataset contains around 120 years of Olympic data represented by approximately two million triples.<br>


The essence of my project was to implement the previously missing SPARQL Expressions by leveraging modern C++ (up to C++20) along with its well established and mature libraries such as STL, Abseil and Boost. The [SPARQL expressions](https://www.w3.org/TR/sparql11-query/#expressions "SPARQL expressions") themselves represent operational functions within the SPARQL Query language and enable users to perform direct operations on the RDF data by combining those expressions.  In the following, I will provide a compact overview about the implementations that have been finalized with this project.


* * *


# <a id="explanation-relevant-terms"></a>2. Relevant Terminology
IRIs and Literals typically represent the input and return values for the implemented functions, making them essential terms and concepts regarding SPARQL expressions. In addition, some expressions, like `TIMEZONE`, handle and operate on [`xsd:dateTime`](https://www.w3.org/TR/2004/REC-xmlschema-2-20041028/#dt-dateTime "`xsd:dateTime`") or `xsd:dayTimeDuration` values. Therefore, here follows a brief explanation of these terms.

## <a id="term-iri-and-uri"></a>2.1 IRI and URI
[URIs](https://www.ietf.org/rfc/rfc3986.txt "URI") and [IRIs](https://www.ietf.org/rfc/rfc3987.txt, "IRI") are similar concepts, URI stands for Uniform Resource Identifier and is a more restricted version of an IRI (Internationalized Resource Identifier). An IRI reference can contain, for example, non-ASCII characters. These identifiers are mostly used in SPARQL engines for resolving the actual interpretation of a value (attached datatype) in a standardized way. A concrete example here is the IRI reference `<http://www.w3.org/2001/XMLSchema#int>`, it makes something clearly and globally identifiable as an integer.<br>
The most commonly used `PREFIX` for IRI references is: `<http://www.w3.org/2001/XMLSchema#>`.<br>
It is used for declaring values such as a `date`, `integer`, `decimal`, `duration` etc. You will see it a lot when interacting with QLever.

## <a id="term-literal"></a>2.2 Literals
Data is typically provided in form of string values by databases or the typed query itself. However, often the intended interpretation should be different to a string with respect to the later on performed operations on those values. For example, we want to be able to interpret `“2000-01-01”` as a date, or `“42”` as an integer, not everything should be considered a simple string. The concept of specific literals helps with differentiating underlying values.
1. Simple Literals: Simple text string like `”simpleStr”` with no attached datatype or language tag.
2. Typed Literals: Text strings with a specified interpretation by attaching a datatype. The general format is `”strValue”^^datatype`, whereby datatype is mostly an IRI or URI reference.
3. Language-tagged Literals: Contain an associated language tag, which is attached with an `@` to directly differentiate simple strings by their language. Exemples: `“hello world”@en` and `“bonjour le monde”@fr`.

## <a id="term-datetime"></a>2.3 xsd:dateTime
`Xsd:dateTime` ([dateTime](https://www.w3.org/TR/2004/REC-xmlschema-2-20041028/#dt-dateTime "dateTime")) represents a general date with a time and an additional optional timezone value. Datetime follows the format by the XML Schema Definition with the general structure: `YYYY-MM-DDThh:mm:ss.sssZ`
- `YYYY`: the year
- `MM` and `DD`: month and day
- `T`: separates the date and time components
- `hh` and `mm`: hours and minutes
- `ss.sss`: seconds with fractional seconds
- `Z`: optional timezone value (typically from `"-14:00"` to `"+14:00"`) <br>

An explicit example following this structure with added corresponding IRI:<br>
`"2000-11-28T03:33:10.000-05:00"^^<http://www.w3.org/2001/XMLSchema#dateTime>`


## <a id="term-daytime-duration"></a>2.4 xsd:dayTimeDuration
`Xsd:dayTimeDuration` ([dayTimeDuration](https://www.ibm.com/docs/ko/i/7.3?topic=types-xsdaytimeduration "dayTimeDuration")) represents a duration value represented by the general format: `PnDTnHnMnS.SSS` and `-PnDTnHnMnS.SSS` (positive and negative durations).
- `P`: indicates a duration
- `nD`: number of days
- `T`: separator
- `nH`: number of hours
- `nM`: number of minutes
- `nS.SSS`: seconds with fractional seconds

Example value following this structure with suitable IRI:
`"P2DT23H23M59.999S"^^<http://www.w3.org/2001/XMLSchema#dayTimeDuration>`<br>

The ability to handle `xsd:dayTimeDuration` values in QLever was introduced with this project, to be specific, with the implementation of expression `TIMEZONE`. QLever stores `xsd:dayTimeDuration` values efficiently by internally converting them to a total millisecond value. When applying operations `==` and `<=>` to `xsd:dayTimeDuration` values, they are performed directly on the bits representing the total milliseconds, which is efficient. 


* * *

# <a id="implemented-epressions"></a>3. Implemented Expressions
Below, the implemented expressions are briefly explained and example queries making use of them are provided. The SPARQL code snippets can be easily copied into [QLever](https://qlever.cs.uni-freiburg.de/ "QLever") (select `Wikidata` as the dataset), try them out!<br>
Hint: The results for the example queries were downloaded as a TSV-file from QLever and inserted in this post as a Markdown table for an improved overview. Thus, the directly visible results may partially differ.

## <a id="datatype-expression"></a>3.1 DATATYPE
A requested expression that returns the IRI (Internationalized Resource Identifier) to a provided literal. The literal can be a simple, typed or language-tagged one.<br>
Given a simple literal containing a string like `“Freiburg”`, `DATATYPE` returns the IRI reference `xsd:string`.<br>
Explicit reference for `xsd:string`: `<http://www.w3.org/2001/XMLSchema#string>`.<br>
Typed literals represent a string like `“42.00”^^xsd:decimal`. Thus, `DATATYPE` returns the reference `xsd:decimal` (`<http://www.w3.org/2001/XMLSchema#decimal>`), or in general the corresponding IRI reference to the primary literal content.<br>
SPARQL 1.1 defines the return IRI as `rdf:langString` for language-tagged literals, such as `“Freiburg”@en`.

```sparql
PREFIX ps: <http://www.wikidata.org/prop/statement/>
PREFIX p: <http://www.wikidata.org/prop/>
PREFIX skos: <http://www.w3.org/2004/02/skos/core#>
PREFIX wd: <http://www.wikidata.org/entity/>
PREFIX wdt: <http://www.wikidata.org/prop/direct/>

SELECT ?name ?birthdate ?podium_finishes ?dt_name
       ?dt_birthdate ?dt_podium_finishes  WHERE {
  # occupation: formula 1 driver
  ?driver wdt:P106 wd:Q10841764 .
  # name of the driver from respective entity node
  ?driver skos:prefLabel ?name .
  # get the brithdate of driver entity
  ?driver wdt:P569 ?birthdate .
  # extension node containing podium data
  ?driver p:P10648 ?podium_node .
  # number of podiums from podium extension node
  ?podium_node ps:P10648 ?podium_finishes .
  # filter for names labeled with en + all drivers in the ouput should have at least 30 podiums 
  FILTER (LANG(?name) = "en" && ?podium_finishes >= 30) .
  # extract the respective datatype for ?name, ?birthdate and ?podium_finishes
  BIND(DATATYPE(?name) AS ?dt_name)
  BIND(DATATYPE(?birthdate) AS ?dt_birthdate) .
  BIND(DATATYPE(?podium_finishes) AS ?dt_podium_finishes)
} 
ORDER BY DESC(?podium_finishes)
LIMIT 5
```
<p style="text-align: center;">Query 1: Extract datatype IRIs by using expression DATATYPE.</p>


| name                   | birthdate              | podium_finishes | dt_name                                                             | dt_birthdate                           | dt_podium_finishes                              |
|------------------------|------------------------|-----------------|---------------------------------------------------------------------|----------------------------------------|------------------------------------------------|
| "Lewis Hamilton"@en    | 1985-01-07T00:00:00Z   | 191             | <http://www.w3.org/1999/02/22-rdf-syntax-ns#langString>             | <http://www.w3.org/2001/XMLSchema#dateTime> | <http://www.w3.org/2001/XMLSchema#double>      |
| "Michael Schumacher"@en| 1969-01-03T00:00:00Z   | 155             | <http://www.w3.org/1999/02/22-rdf-syntax-ns#langString>             | <http://www.w3.org/2001/XMLSchema#dateTime> | <http://www.w3.org/2001/XMLSchema#double>      |
| "Sebastian Vettel"@en  | 1987-07-03T00:00:00Z   | 122             | <http://www.w3.org/1999/02/22-rdf-syntax-ns#langString>             | <http://www.w3.org/2001/XMLSchema#dateTime> | <http://www.w3.org/2001/XMLSchema#double>      |
| "Alain Prost"@en       | 1955-02-24T00:00:00Z   | 106             | <http://www.w3.org/1999/02/22-rdf-syntax-ns#langString>             | <http://www.w3.org/2001/XMLSchema#dateTime> | <http://www.w3.org/2001/XMLSchema#double>      |
| "Kimi Räikkönen"@en    | 1979-10-17T00:00:00Z   | 103             | <http://www.w3.org/1999/02/22-rdf-syntax-ns#langString>             | <http://www.w3.org/2001/XMLSchema#dateTime> | <http://www.w3.org/2001/XMLSchema#double>      |

<p style="text-align: center;">Result 1: The five F1 drivers with the most podium finishes and the extracted datatypes.</p>

## <a id="strdt-and-strlang-expression"></a>3.2 STRDT and STRLANG
`STRDT` and `STRLANG` offer similar functionalities. Both implementations enable users to intuitively assign additional information to a simple literal: `STRDT` assigns a datatype, while `STRLANG` assigns a language tag.<br>

`STRDT` assigns an explicit IRI string to a simple literal, this can be useful for queries where for example filtering with respect to datatypes is relevant. `STRDT(“3.141”,  xsd:decimal)` would result in a literal that stores `“3.141”^^<http://www.w3.org/2001/XMLSchema#decimal>`. A more explicit declaration is `STRDT(“42”, <http://www.w3.org/2001/XMLSchema#int>)`, resulting analogous in a literal containing `“42”^^<http://www.w3.org/2001/XMLSchema#int>`. QLever grants certain freedoms when it comes to IRIs, we don’t check formally if the format of the provided IRI is correct. Thus, `“valid”` (or similar simple string values) provided as an IRI string would result in an literal with `^^<valid>` as an added datatype.<br>

`STRLANG` combines a simple literal with its respective, as a string declared, language tag. With `STRLANG(“Freiburg”, “de-LATN-DE”)`, a literal containing the value `“Freiburg”@en-LATN-de` is created. The current implementation only verifies the format of the declared language tag by using the simple Regex `[a-zA-Z]+(-[a-zA-Z0-9]+)*`, hence we don't check for general correctness. <br>

Example query using `STRLANG`: See Query 4.

## <a id="lang-and-langmatches-expression"></a>3.3 LANG and LANGMATCHES
The expression `LANG` can be functionally considered the inverse to `STRLANG`. `LANG` simply returns the language tag as a string to a passed literal. For `LANG(STRLANG(“Freiburg”, “de-LATN-DE”))`, the result would be a simple literal that contains `“de-LATN-DE”`. If the provided literal has no underlying language tag, it is resolved as `""`. <br>

`LANGMATCHES` checks if a provided language tag is contained within a specified language range, this procedure follows concept of [Basic Filtering](https://www.ietf.org/rfc/rfc4647.txt "Basic Filtering") (Section 3.3.1). If the language tag is contained within the defined range, `LANGMATCHES` returns `true`, else `false`. Given that a boolean value is returned, `LANGMATCHES` is quite handy because it can be easily used within the expression `FILTER` (see Query 2).<br>
It is important to note, that `FILTER(LANGMATCHES(LANG(?x), “en”))` is not equivalent to `FILTER(LANG(?x) = “en”)`. This is because we resolve with respect to a language range. If `?x` would represent here the literal `“Freiburg”@en-US`,  `FILTER(LANG(?x) =  “en”)` would drop `“Freiburg”@en-US`  for the output, while with `FILTER(LANGMATCHES(LANG(?x), “en”))` it would be kept.<br>

```sparql
PREFIX skos: <http://www.w3.org/2004/02/skos/core#>
PREFIX wd: <http://www.wikidata.org/entity/>
PREFIX wdt: <http://www.wikidata.org/prop/direct/>

SELECT ?president ?name ?birthdate WHERE {
  # occupation: president of the United States
  ?president wdt:P39 wd:Q11696 .
  # date of brith
  ?president wdt:P569 ?birthdate .
  # attached name to entity
  ?president skos:prefLabel ?name .
  # retrieve the entites where the label of the name matches the language range "en"
  FILTER (LANGMATCHES(LANG(?name), "en"))
}
# order output by ascending birthdate
ORDER BY ASC(?birthdate)
LIMIT 5
```
<p style="text-align: center;">Query 2: Query using LANGMATCHES and LANG.</p>

| president                                 | name                       | birthdate              |
|-------------------------------------------|----------------------------|------------------------|
| <http://www.wikidata.org/entity/Q23>      | "George Washington"@en-gb  | 1732-02-22T00:00:00Z   |
| <http://www.wikidata.org/entity/Q23>      | "George Washington"@en-ca  | 1732-02-22T00:00:00Z   |
| <http://www.wikidata.org/entity/Q23>      | "George Washington"@en     | 1732-02-22T00:00:00Z   |
| <http://www.wikidata.org/entity/Q11806>   | "John Adams"@en            | 1735-10-30T00:00:00Z   |
| <http://www.wikidata.org/entity/Q11806>   | "John Adams"@en-ca         | 1735-10-30T00:00:00Z   |

<p style="text-align: center;">Result 2: The NAME column clearly indicates that LANGMATCHES was used.</p>

## <a id="now-expression"></a>3.4 NOW
The expression `NOW`, as the name already suggests, returns the point in time at which the query is executed in form of a `xsd:dateTime`. By definition, the returned `xsd:dateTime` value is the same throughout the query. The implementation uses the Abseil library and sets exactly once a (string) variable to the current point in time with `absl::FormatTime("%Y-%m-%dT%H:%M:%E3S%Ez", absl::Now(), absl::LocalTimeZone())`. This value is in the following used for all evaluations of `NOW` while executing the query. <br>

Example query using `NOW`: See Query 3.

## <a id="timezone-and-tz-expression"></a>3.5 TIMEZONE and TZ
The expression `TIMEZONE` takes a `xsd:dateTime` value as an argument, and returns the extracted timezone from this object in the corresponding string format of a `xsd:dayTimeDuration` value. `Xsd:dateTime` objects don't necessarily contain a timezone, e.g. `”2024-01-01T00:01:11.12.435”^^xsd:dateTime`. For such datetime values without a timezone,  the `undefined` value is returned. <br>
For `xsd:dateTime` objects containing a timezone value,  we return a `xsd:dayTimeDuration`. Concrete example: `TIMEZONE(”2024-01-01T00:01:11.12.435-10:00”^^xsd:dateTime)` returns `-PT10H`, a `xsd:dayTimeDuration` value. <br>
There is in addition a special timezone value `Z` which represents `UTC+0`. For provided `xsd:dateTime` objects with timezone `Z`, `PT0S` is returned. <br>

`TZ` retrieves the timezone of a passed `xsd:dateTime` argument as well. However, unlike expression `TIMEZONE`, which returns an underlying `xsd:dayTimeDuration` object, `TZ` directly returns the timezone value from the datetime as a string (e.g. `"+02:00"`).

```sparql
PREFIX skos: <http://www.w3.org/2004/02/skos/core#>
PREFIX wd: <http://www.wikidata.org/entity/>
PREFIX wdt: <http://www.wikidata.org/prop/direct/>

SELECT ?city ?city_name (TZ(?current_time) AS ?tz_timezone)
       # extract the timezone once with TIMEZONE and once with TZ for comparison
       # from ?current_time (xsd:dateTime) 
       (TIMEZONE(?current_time) AS ?timezone_timezone) WHERE {
  # big city statement
  ?city wdt:P31 wd:Q1549591 .
  # city located in Germany
  ?city wdt:P131 wd:Q183 .
  # get the name of the city
  ?city skos:prefLabel  ?city_name .
  # population of the city
  ?city wdt:P1082 ?population .
  # city name should match "de" tags and its population should be above 3.000.000
  FILTER(LANGMATCHES(LANG(?city_name), "de") && ?population > 3000000) .
  # bind time of execution to variable ?current_time
  BIND(NOW() AS ?current_time)
}
```
<p style="text-align: center;">Query 3: Query using NOW, TIMEZONE and TZ.</p>

| ?city                                           | ?city_name     | ?tz_timezone | ?timezone_timezone |
|------------------------------------------------|----------------|--------------|---------------------|
| <http://www.wikidata.org/entity/Q64>           | "Berlin"@de    | "+02:00"     | PT2H                |
| <http://www.wikidata.org/entity/Q64>           | "Berlin"@de-at | "+02:00"     | PT2H                |
<p style="text-align: center;">Result 3: Result indicates that TZ returns a simple string, while TIMEZONE returns an actual xsd:dayTimeDuration value.</p>

## <a id="struuid-and-uuid-expression"></a>3.6 UUID and STRUUID
These expressions return a fresh Universally Unique Identifier in the common 128-bit format as defined by [RFC4122](https://www.ietf.org/rfc/rfc4122.txt "RFC4122"). They are designed to be globally unique within a system. The implementation makes use of the Boost library with [`boost::uuid`](https://www.boost.org/doc/libs/1_85_0/libs/uuid/doc/uuid.html "  boost::uuid"), which directly enables the creation of UUIDs and their subsequent conversion to string values. UUIDs are usually used within large databases and networks to clearly identify contained entities, hence the SPARQL 1.1 standard regards the creation of UUIDs as relevant. <br>
Fundamentally, `STRUUID` and `UUID` are equal in their underlying implementation. The difference is that `STRUUID` returns a literal with the underlying plain UUID, while `UUID` returns an UUID URN schemed IRI reference. The UUID itself results with correct formatting in a 36 character string representation.

```sparql
PREFIX wd: <http://www.wikidata.org/entity/>
PREFIX skos: <http://www.w3.org/2004/02/skos/core#>
PREFIX wdt: <http://www.wikidata.org/prop/direct/>
SELECT ?name ?uuid ?struuid WHERE {
  # professional golfer
  ?golfer wdt:P106 wd:Q11303721 .
  # get the name of the golfer
  ?golfer skos:prefLabel ?name .
  # filter for "de-AT" language tagged name
  FILTER(LANGMATCHES(LANG(?name), "de-AT")) .
  # filter for Tiger Woods
  FILTER(?name = STRLANG("Tiger Woods", "de-AT")) .
  # assign each output row a unique identifier (urn uuid and string uuid)
  BIND(UUID() AS ?uuid) .
  BIND(STRUUID() AS ?struuid)
}
```
<p style="text-align: center;">Query 4: Make each result row clearly indentifiable by UUID assignment.</p>

| ?name          | ?uuid                                      | ?struuid                                |
|----------------|--------------------------------------------|----------------------------------------|
| "Tiger Woods"@de-at | <urn:uuid:63dc1082-6fed-4bd0-b218-447d2e083157> | "1d795abc-6545-459d-8762-b1ec3a676c83" |
<p style="text-align: center;">Result 4: Output with UUID, once represented as a urn and once as a simple string. The UUID values are different with every execution!</p>

## <a id="hash-expression"></a>3.7 Hash Expressions
The SPARQL 1.1 standard considers the hash operations `MD5`, `SHA1`, `SHA256`, `SHA384` and `SHA512` as relevant. Passed a simple literal, these functions calculate the checksum for the underlying string value and return it in the following as a hex digit string. The implementation uses the OpenSSL library with its convenient EVP interface.

```sparql
PREFIX skos: <http://www.w3.org/2004/02/skos/core#>
PREFIX wd: <http://www.wikidata.org/entity/>
PREFIX wdt: <http://www.wikidata.org/prop/direct/>

SELECT ?president ?name ?birthdate ?md5 ?sha1 ?sha256 ?sha384 ?sha512 WHERE {
  # occupation: president of the United States
  ?president wdt:P39 wd:Q11696 .
  # date of brith
  ?president wdt:P569 ?birthdate .
  # attached name to entity
  ?president skos:prefLabel ?name .
  # retrieve the entites where the label of the name matches the language range "en"
  FILTER (LANGMATCHES(LANG(?name), "en")) .
  # create with each hash expression a hash for ?name (George Washington)
  BIND(MD5(?name) AS ?md5) .
  BIND(SHA1(?name) AS ?sha1) .
  BIND(SHA256(?name) AS ?sha256) .
  BIND(SHA384(?name) AS ?sha384) .
  BIND(SHA512(?name) AS ?sha512)
}
# order output by ascending birthdate
ORDER BY ASC(?birthdate)
LIMIT 1
```
<p style="text-align: center;">Query 5: Query hashing ?name with every available hash implementation.</p>

| president                            | name                         | birthdate           | md5                              | sha1                                     | sha256                                                          | sha384                                                                                       | sha512                                                                                                                                                                   |
|--------------------------------------|------------------------------|---------------------|----------------------------------|------------------------------------------|----------------------------------------------------------------|------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| <http://www.wikidata.org/entity/Q23> | "George Washington"@en-gb    | 1732-02-22T00:00:00Z| 6c7a497506886d03de03a412f14d7747 | e89f0a075045d5ad9965da19d18dd14d97d86484 | 7b5561e37d4a2fcbf132ce0946e7ddeb8183ea4461c96a3cefb4f4296e1a94f8 | 1e7795ae343edad0972ae6aa8b146ca10074117ebdec1bc51da5d72cc48441be4fc93541c9d79275d4396b405aa421ea | b5a29220138796638bc4dc6e79e44d9d84e6eceb5e59ab08f66259752f1ae3749e59de536905605024adc6d2200b6d583af4d300fa517fa90cf337512c7915d5 |

<p style="text-align: center;">Result 5: Contains for each hash function the hash hex string of George Washington.</p>

* * *

# <a id="conclusion"></a>4. Conclusion
The implementations work well and can be used and tested with the official [QLever](https://qlever.cs.uni-freiburg.de/ "QLever") browser interface. However, given that most implementations have been implemented on already exisiting expression infrastructure in combination with the fact that some expressions heavily rely on passing and comparing strings internally, there is still room for optimization.<br>
Mentionable expressions which could be significantly optimized are `LANGMATCHES` and `DATATYPE`, especially under the assumption that they are likely to be used often. Within the rather costly implementation of `LANGMATCHES`, the strings of the language tag and the language range are compared explicitly. In the case of `DATATYPE`, all string values must be retrieved from the disk, which could be optimized by directly encoding the datatype into the internal `ID` of the respective value.
