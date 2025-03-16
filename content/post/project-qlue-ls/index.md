---
title: "Qlue-ls a SPARQL language server"
date: 2025-01-12T12:43:04+02:00
author: "Ioannis Nezis"
authorAvatar: "img/profile.png"
tags: ["SPARQL", "lsp", "wasm", "rust"]
categories: []
image: "img/cover.png"
---

Modern developer environments are way more capable than simple text editors.
They provide domain-specific tools to improve the user experience.
They give hints, suggest changes and completions and more.
In this article, we will take a look behind the curtains and build language support for *SPARQL*,
a query language for knowledge graphs.

<!--more-->

You can find the source code in my [GitHub repository](https://github.com/IoannisNezis/sparql-language-server).  
And a live demo on [qlue-ls.com](https://qlue-ls.com/).


# TL;DR

I built a [sparql-language-server](https://github.com/IoannisNezis/Qlue-ls) from scratch in [Rust](https://www.rust-lang.org/), powered by [tree-sitter](https://tree-sitter.github.io/tree-sitter/).
To showcase the language server I built a [web editor](https://qlue-ls.com/) using [Monaco](https://microsoft.github.io/monaco-editor/).
To run the language server within the browser, I used [WebAssembly](https://webassembly.org/)

# Content

- [Motivation](#motivation)
- [Goal](#goal)
- [The Language Server Protocol ](#the-language-server-protocol)
    - [JSON-RPC](#json-rpc)
    - [Document Synchronization](#document-synchronization)
    - [Capabilities](#capabilities)
- [Implementation](#implementation)
    - [speakingJSON-RPC](#speaking-json-rpc)
    - [Parser: the Engine under the Hood](#parser-the-engine-under-the-hood)
        - [Tree-sitter](#tree-sitter)
    - [Implemented Capabilities](#implemented-capabilities)
        - [Formatting](#formatting)
        - [Hover](#hover)
        - [Diagnostics](#diagnostics)
        - [Code Actions](#code-actions)
        - [Completion Suggestions](#completion-suggestions)
- [Using the Language server](#using-the-language-server)
    - [Neovim](#neovim)
        - [Installing the Programm](#installing-the-programm)
        - [Attaching](#attaching)
    - [VS-code](#vs-code)
    - [The Browser](#the-browser)
        - [WebAssembly](#webassembly-wasm)
            - [Tree-Sitter in WebAssembly](#tree-sitter-in-webassembly)
        - [The Editor](#the-editor)
        - [TextMate](#textmate)
        - [Plugging everything together](#plugging-everything-together)
- [How does Qlue-ls compare against other software?](#how-does-qlue-ls-compare-against-other-software)
    - [Qlue-ls vs sparql-formatter](#qlue-ls-vs-sparql-formatter)
- [Future work](#future-work)
    - [Stronger Parser](#stronger-parser)
    - [Query Sparql endpoint](#query-sparql-endpoint)
    - [Enhance existing features](#enhance-existing-features)
- [Acknowledgements](#acknowledgements)

# Motivation

The problem of providing language support to developers is very old.
In the past, domain-specific development environments were very common.

| Domain    | development environment |
| --------- | ----------------------- |
| Java      | Eclipse                 |
| Microsoft | Visual Studio           |
| C/C++     | Turbo C++               |
| Python    | PyCharm                 |
| R         | RStudio                 |
| LaTeX     | TeXworks or Overleave   |

These programs contain source-code editors, but also provide a suite of integrated tools that support the development process of their respective domains.
That's why they are also referred to as IDE's (**I**ntegrated **D**evelopment **E**nvironment).
While these development environments still dominate, modern development environments seem to go into a different direction.

Some of the new kids on the block are: [neovim](https://neovim.io/), [vscode](https://code.visualstudio.com/) or [sublime text](https://www.sublimetext.com/).
They all are **general purpose** code editors that have a open-source plugin ecosystem and allow a personalized customization. A core maintainer of Neovim,
[TJ DeVries](https://github.com/tjdevries), calls them PDE's (**P**ersonalized **D**evelopment **E**nvironment), although I don't think it caught on yet.

Long story short: Language support in these PDE's is not built-in, but provided via an extension.
This is made possible by a protocol published by Microsoft in 2016: The **L**anguage **S**erver **P**rotocol (LSP).
It enables the editor (LSP-Client) and the Language support program (LSP-Server or Language Server) to be separated into two independent components.

![](img/language-server-sequence.png)
[^2]

A key advantage of this architecture is its reusablitity.  
The language support has to be written only once and not over and over again for every development tool.

![](img/lsp-languages-editors.png)[^7]

# Goal

My goal is to create a language server for [SPARQL](https://www.w3.org/TR/sparql11-query/#rQueryUnit).
The language server should be able to **format** queries, give **diagnostic** reports and suggest **completions**.
To work in the [Qlever UI](https://qlever.cs.uni-freiburg.de/), the language server should be accessible from an editor which runs in the browser.

# The Language Server Protocol

Let's talk briefly about the protocol.
It's built on top of [JSON-RPC](https://www.jsonrpc.org/specification), a [*JSON*](https://de.wikipedia.org/wiki/JSON) based protocol that enables, as the name suggests, inter-process communication.  
This means that the development tool (in our case the editor) and the language server run in two separate processes and communicate asynchronously via **JSON-RPC**.

## JSON-RPC

I will just give you a brief introduction, if you want to know more, you can read the [specification](https://www.jsonrpc.org/specification#id1).

A normal JSON-RPC request has a `id`, `method` and `params` field.  
The `method` is the name of the invoked operation, the `params` field contains the optional parameters for this invoked operation.

A normal JSON-RPC response has a `id` and `result` field.  
The `id` has to be the same as the `id` of the corresponding request. This enables asynchronous communication.  
The `params` contain the result of the operation, if present.

{{< notice example >}}
 Request: `{"jsonrpc": "2.0", "method": "add", "params": [21, 21], "id": 1}`
 Response: `{"jsonrpc": "2.0", "result": 42, "id": 1}`
{{< /notice >}}

There are also notifications[^4] and error responses, but we will omit them for now.

## Document synchronization

For the language server to be able to do anything, it has to "*see*" the workspace.

The LSP-specification defines 3 [Document Synchronization](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_synchronization) methods for this purpose:
 - [`textDocument/didOpen`](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_didOpen)
 - [`textDocument/didChange`](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_didChange)
 - [`textDocument/didClose`](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_didClose)

These are mandatory to implement (for clients).  
Whenever a document is being opened, changed or closed, the client is sending the information to the server via these methods.

The [textDocument/didChange](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_didChange) notification[^4] supports full and incremental synchronization.
The server and client negotiate this during [initialization](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#initialize).

{{< notice info >}}
Incremental synchronization was more diffucult to implement than expected.
This is mainly because of the translation between different encodings.
Editors give the position in the text-document (row, column) based on the utf-16 string representation.
While the chars themself are encoded in UTF-8.
Of course, UTF-8 is a variable length encoding, so different characters can have different byte-sizes.

![](img/encodings.png)

This was a bit confusing to get right.
{{< /notice >}}

Through these messages, the language server has a "mirrored" version of the editor state.

{{< notice example >}}
 Here is an example for a incremental **textDocument/didChange** notification.
```json
 {
	"params": {
		"contentChanges": [
			{
				"text": "42",
				"range": {
					"end": {
						"line": 3,
						"character": 22
					},
					"start": {
						"line": 3,
						"character": 16
					}
				}
			}
		],
		"textDocument": {
			"uri": "file:\/\/\/home\/ianni\/code\/sparql-language-server\/example.sparql",
			"version": 227
		}
	},
	"jsonrpc": "2.0",
	"method": "textDocument\/didChange"
}
```
{{< /notice >}}

## Capabilities

When initialization and synchronization work, the real fun begins.  
Now we can implement complex language features and provide actual smarts to the editor,
as long as both the client and server support the capability.

Here is an **incomplete** list of language feature capabilities that made it into the specification

| Capability                                                                                                                                         | Effect                                                        | State of implementation |
| -------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------- | ----------------------- |
| [Go to declaration](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_declaration)          | Jump to the declaration of a symbol                          | Not planned             |
| [Go to definition](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_definition)            | Jump to the definition of a symbol                           | Not planned             |
| [Document highlight](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_documentHighlight)   | Highlight  all references to a symbol                         | Planned                 |
| [Document link]()                                                                                                                                  | Handle links in a document                                    | Planned                 |
| [Hover](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_hover)                            | Show information about the symbol hovered above                     | In progess              |
| [Folding range](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_hover)                    | Identify  foldable ranges in the document.                  | Not planned             |
| [Document symbols](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_documentSymbol)        | Identify all symbols in a document                            | Not planned             |
| [Inline value](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_inlineValue)               | Show values in the editor                                     | Not Planned             |
| [Completion proposals](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_completion)        | Give completion proposals to the user                         | In progress             |
| [Publish diagnostics](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_publishDiagnostics) | Send information like hints, warnings or errors to the editor | In progress             |
| [Pull diagnostics](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_pullDiagnostics)       | Request information like hints, warnings or errors              | In progress             |
| [Code action](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_codeAction)                 | Suggest changes                                               | In progress             |
| [Formatting](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_formatting)                  | Format the whole document                                     | Done                    |
| [Range formatting](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_rangeFormatting)       | Format the provided range in a document                       | Not planned             |
| [Rename](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_rangeFormatting)                 | Rename a symbol                                               | Planned                 |

# Implementation

Let's talk about what I actually did.

I chose to use [Rust](https://www.rust-lang.org/) for this project since it's fancy and I like shiny things.  
Rust is the most admired programming language in the [Stack overflow developer survey 2024](https://survey.stackoverflow.co/2024/technology#2-programming-scripting-and-markup-languages),
and I was curious to find out why. After this project, I can confirm that Rust is a brilliant language, but the learing curve is quite steep.

The error handling, incredibly smart compiler, functional approach and rich ecosystem enable a smooth developing experience.
That being said, the very strict compiler makes it hard to get stuff done quickly, however the resulting code is a lot more robust.

Here is the module structure of my crate[^5]:

![](img/code_structure.svg)

## speaking JSON-RPC

Okay first things first, we need to speak **JSON-RPC**.  
After that we can implement some tool to analyze SPARQL queries.  
When we get the analysis tool running, we can use it to provide some language features.

Assume we set up the editor (client) to connect to our language server.  
It will send an UTF-8 byte-stream. We need to interpret the bytes and respond.

The first message will look like something like this:

```json
{
	"jsonrpc": "2.0",
	"id": 1,
	"method": "initialize",
	"params": {
		"capabilities": {...},
		"clientInfo": {
			"name": "Neovim",
			"version": "0.10.2+v0.10.2"
		},
		"processId": 6369,
		...
	}
}
```

For each type of message, I built a set of corresponding structs:

```rust
#[derive(Serialize, Deserialize, Debug, PartialEq)]
pub struct Message {
    pub jsonrpc: String,
}

#[derive(Serialize, Deserialize, Debug, PartialEq)]
pub struct RequestMessageBase {
    #[serde(flatten)]
    pub base: Message,
    /**
     * The request id.
     */
    pub id: RequestId,
    /**
     * The method to be invoked.
     */
    pub method: String,
}

#[derive(Serialize, Deserialize, Debug, PartialEq, Clone)]
#[serde(untagged)]
pub enum RequestId {
    String(String),
    Integer(u32),
}

#[derive(Debug, Serialize, Deserialize, PartialEq)]
pub struct InitializeRequest {
    #[serde(flatten)]
    pub base: RequestMessageBase,
    pub params: InitializeParams,
}

#[derive(Debug, Serialize, Deserialize, PartialEq)]
#[serde(rename_all = "camelCase")]
pub struct InitializeParams {
    pub process_id: ProcessId,
    pub client_info: Option<ClientInfo>,
    #[serde(flatten)]
    pub progress_params: WorkDoneProgressParams,
}

#[derive(Debug, Serialize, Deserialize, PartialEq)]
#[serde(untagged)]
pub enum ProcessId {
    Integer(i32),
    Null,
}

#[derive(Debug, Serialize, Deserialize, PartialEq)]
pub struct ClientInfo {
    pub name: String,
    pub version: Option<String>,
}
```

For **se**rializing and **de**serializing I used [serde](https://serde.rs/).  

{{< notice note >}}
  In Rust the notion of "inheritance" does not exist. It uses "traits" to define shared behavior.  
  I solved this issue with `#[serde(flatten)]`, which inlines the data from a struct into a parent struct.  
  Another issue was the naming convention.  
  In JSON-RPC the fields are written in *camelCase*, but Rust uses *snake_case*.  
  Serde also offers a solution for that: the `#[serde(rename_all = "camelCase")]` annotation.
{{< /notice >}}

This is basically how I read and write messages.
I defined structs for the basic [lifecycle massages](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#initialize):

| message                                                                                                               | sender | type         | effect                                         |
| --------------------------------------------------------------------------------------------------------------------- | ------ | ------------ | ---------------------------------------------- |
| [initialize](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#initialize)  | client | request      | initialize connection, comunicate capabilities |
| [initalized](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#initialized) | client | notification | signals reception of initialize response        |
| [shutdown](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#shutdown)      | client | request      | shutdown server, don't exit                     |
| [exit](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#exit)              | client | notification | exit the server process                        |

Then I defined the basic structs for  [document synchronization](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_synchronization)

| message                                                                                                                                      | sender | type         | effect                              |
| -------------------------------------------------------------------------------------------------------------------------------------------- | ------ | ------------ | ----------------------------------- |
| [textDocument/didOpen](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_didOpen)     | client | notification | signals newly opened text document   |
| [textDocument/didChange](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_didChange) | client | notification | signals changes to a text document   |
| [textDocument/didClose](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_didClose)   | client | notification | signals closed text document |

With these messages defined, we can open and close a connection to a client and keep a synchronized state of the client's documents.

## Parser: the Engine under the hood

Okay, now we want to build some tools to analyze the given text,
and provide some smarts.

First we need to "understand" the given text.
Understanding arbitrary text is quite the challenge, and we only recently made some advancements in that area.
Luckily, all SPARQL queries follow a strict structure, called a grammar, which is defined in its [specification](https://www.w3.org/TR/sparql11-query/#rQuery).
A grammar is basically a set of production rules that map nonterminal-symbols to other nonterminal-symbols or terminal-symbols. Every valid SPARQL query can be produced by applying those rules until only terminal symbols are left.
For some grammars, we can build a program that reconstructs which production rules got used to produce a given string. Such a program is called **parser**. The result of a parser, the rules that got used, is called **syntax tree**.

Here is an example:

**Query**:
```sparql
SELECT * WHERE {}
```

**Syntax tree:**

![](img/SyntaxTree.svg)

Here is the same syntax tree in a textual representation:
```lisp
(unit ; [0, 0] - [1, 0]
  (select_query ; [0, 0] - [0, 17]
    (select_clause ; [0, 0] - [0, 8]
      "SELECT" ; [0, 0] - [0, 6]
      "*") ; [0, 7] - [0, 8]
    (where_clause ; [0, 9] - [0, 17]
      "WHERE" ; [0, 9] - [0, 14]
      (group_graph_pattern ; [0, 15] - [0, 17]
        "{" ; [0, 15] - [0, 16]
        "}")))) ; [0, 16] - [0, 17]
```

### Tree-sitter

Usually parsers get to parse complete strings that are valid.
When you give them invalid input they react less happy (symbolic image below).

![](img/resiliant-parsers.png)
[^3]

In our use case, the text we handle is incomplete most of the time.
So we need a parser that is resilient to errors.

The big-boy language servers like [rust-analyser](https://github.com/rust-lang/rust-analyzer) use customized [resilient LL Parsers](https://matklad.github.io/2023/05/21/resilient-ll-parsing-tutorial.html).
But since I wanted to get on the road quickly, I chose [tree-sitter](https://tree-sitter.github.io/tree-sitter/) (for now... ;)).
Tree-sitter, at its core, is a parser generator. It generates error resilient parsers.
It is most commonly used in editors like [neovim](https://neovim.io/) or emacs[^6] to highlight text.

Parser generators take a grammar and generate a fully functioning parser.
I found a [sparql-tree-sitter](https://github.com/GordianDziwis/tree-sitter-sparql) grammar by [GordianDziwis](https://github.com/GordianDziwis).
It had a few minor issues and was a bit dusty.
So I [forked](https://github.com/GordianDziwis) it and it and changed a few things to work better with Rust.

If you want to take a close look at the parser:

1. Clone the grammar-repository:

```bash
git clone https://github.com/IoannisNezis/tree-sitter-sparql.git
cd tree-sitter-sparql
```

2. Install the [tree-sitter CLI](https://tree-sitter.github.io/tree-sitter/creating-parsers#installation)
3. Start the playground web-app:
```bash
tree-sitter playground
```

The generated parser is written in C.
For some C-reasons I won't get into right now, tree-sitter can generate rust bindings that allow us to call the C-functions from our rust-program (something I will regret later).
It provides some functions to parse and navigate the resulting concrete-syntax-tree.

A cool feature of tree-sitter is [queries](https://tree-sitter.github.io/tree-sitter/using-parsers#pattern-matching-with-queries).
A tree-sitter-query is built out of one or more patterns. Each pattern is an [S-expression](https://en.wikipedia.org/wiki/S-expression).
Tree-sitter can match these patterns against the syntax tree and return all matches.

{{< notice example >}}
Let's say we want to find all *triples* in a group-graph-pattern that have a predicate that match "pre:p1":
The query could look like:
```sparql
SELECT * WHERE {
    ?s pre:p2 "object1" .
    ?s2 pre:p1 "object2"
}
```
My tree-sitter parser would generate this parse-tree:
```lsip
(unit
  (SelectQuery
    (SelectClause)
    (WhereClause
      (GroupGraphPattern
        (GroupGraphPatternSub
          (TriplesBlock
            (TriplesSameSubjectPath
              subject: (VAR)
              (PropertyListPathNotEmpty
                predicate: (Path
                  (PathSequence
                    (PathEltOrInverse
                      (PathElt
                        (PathPrimary
                          (PrefixedName
                            (PNAME_NS
                              (PN_PREFIX))
                            (PN_LOCAL)))))))
                (ObjectList
                  object: (RdfLiteral
                    value: (String
                      (STRING_LITERAL))))))
            (TriplesSameSubjectPath
              subject: (VAR)
              (PropertyListPathNotEmpty
                predicate: (Path
                  (PathSequence
                    (PathEltOrInverse
                      (PathElt
                        (PathPrimary
                          (PrefixedName
                            (PNAME_NS
                              (PN_PREFIX))
                            (PN_LOCAL)))))))
                (ObjectList
                  object: (RdfLiteral
                    value: (String
                      (STRING_LITERAL))))))))))))
```

And here is the query:
```lisp
(TriplesSameSubjectPath
  (VAR)
  (PropertyListPathNotEmpty
    (Path) @path (#eq? @path "pre:p1"))) @triple
```
`@path` and `@triple` are captures and store the nodes that match the pattern.
{{< /notice >}}

#### How resilient is tree-sitter?

Very resilient.
Tree-sitter recovers from basically anything and conserves information from very incomplete input.

The problem is that the way it conserves information is not optimal for our use case.

Tree-sitter produces GLR parsers.
It  nondeterministically explores many different possible LR (bottom-up) parses.
Then it chooses the "the best one".
In the words of [Alex Kladov](https://github.com/matklad), the creator of [rust-analyzer](https://github.com/rust-lang/rust-analyzer), this leads to the following behavior:
>(...) Tree-sitter \[can\] recognize many complete valid small fragments of a tree, but it might have trouble
> assembling them into incomplete larger fragments.

This is very useful for the use case of syntax highlighting. You want to highlight as many tokens as possible, the larger context in which they appear is often not so important.

In our use-case, however, it's the other way around.

## Implemented Capabilities

Okay, now that you may have an idea of the fundamental mechanisms,
let's talk about the features, besides the lifecycle and synchronization that I implemented.

{{< notice warning >}}
The implemented features are just a **proof of concept**!
There are many features that could and should be added.
But that takes a lot of time to get right.
And, frankly, also requires a stronger parser.
This should just give you an idea of what's possible.
{{< /notice >}}

### Formatting

When the client sends a `textDocument/formatting` request, the server returns a
list of [`TextEdit`](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textEdit)'s.
A `TextEdit` is a incremental change, composed of a range and a string.
The client then applies the changes, thus formatting the document.

![](img/TextEdits.svg)

#### Formatting Algorithm

After a few iterations, I came up with the following algorithm.

---

**Input**: string (sequence of characters)  
**Output**: sequence of textedits

---

**Step 1**: Parse  
Use the [parser](#parser-the-engine-under-the-hood) to compute a sytax tree.  

![](img/SyntaxTree.svg)

**Step 2**: Separate  
For each kind of node, define a separator string. Then compute edits to speparate its child with this separator.

![](img/FormattingSeparation.svg)

**Step 3**: Augment  
For each node, compute augmentation edits. Insert these before, after, or in a node.  

![](img/FormattingAugmentation.svg)

---

{{< notice "note" >}}
In my implementation, **Step 2** and **Step 3** are executed in a recursive manner.  
When traversing the tree, a indentation level is parsed down the tree and increased based on the kind of the node.  

**Step 2** and **3** are, in their essence, a [catamorphism](https://en.wikipedia.org/wiki/Catamorphism)
from a syntax tree to a sequence of edits.  
Here, of course, a syntax tree is a [endofunctor](https://en.wikipedia.org/wiki/Functor#endofunctor) in the category 
of types and functions, but I am digressing.
{{< /notice >}}

---

**Step 4**: Consolidate  
Sort edits by starting point and consolidate consecutive edits.  
Optionally also remove redundant edits.

![](img/FormattingConsolidation.svg)

---

The SPARQL grammar is quite large (138 rules), so implementing this in detail was a bit tedious.  

#### Results

Here are a few examples to give you an idea.  
If want to get a better understanding of the behaviour of this formatter,  
try it out in the [demo](https://qlue-ls.com).

```sparql
BASE <http://...>
PREFIX namespace1: <iri>
PREFIX namespace12: <iri>
SELECT ?s ?p (MAX (?o)  AS ?max_a )
FROM <...>
FROM <...>
WHERE {
    {
        ?s namespace1:p1 ?o ;
          namespace12:p2 "dings" .
        ?other <abc> 32 .
        FILTER ((?other * 4) > 6 || ?o = "bar")
    }
    UNION {
        {
            SELECT * WHERE {
                ?a ?b ?c .
                BIND (?c + 3 AS ?var)
            }
            GROUP BY ?a
            HAVING ?b > 9
            ORDER BY DESC(?a)
            LIMIT 100
            OFFSET 10
        }
    }
}
```

```sparql
SELECT * WHERE {
    wd:Q11571 p:P166 [ pq:P585 ?date ] .
    wd:Q11572 p:P166 [
        pq:P585 ?date ;
        <...> ?other
    ]
    wd:Q11572 p:P166 []
}
```

```sparql
SELECT * WHERE {
    ?a ?b ",,," FILTER (1 IN (1, 2, 3))
}
```

```sparql
SELECT * WHERE {
    ?a <iri>/^a/(!<>?)+ | (<iri> | ^a | a) ?b .
}
```

I added some optional formatting, like aligning prefix declarations and predicates.
Here is an example of a query with every non default option:

```sparql
prefix n1:     <...>
prefix n12:    <...>
prefix n123:   <...>
prefix n1234:  <...>
prefix n12345: <...>

select * 
where {
  ?var n1:p "ding" ;
    n12:p "foo" ;
    n123:p "bar" ;
    n1234:p "baz" ;
    n12345:p "out of filler" ;
}
```

Here is the full default format configuration:

```toml
[format]
align_predicates = true
align_prefixes = false
separate_prolouge = false
capitalize_keywords = true
insert_spaces = true
tab_size = 2
where_new_line = false
filter_same_line = true
```

#### Formatting comments

This algorithm fails when comments appear in the input string.  
For example, the following query

```sparql
SELECT ?a # comment
WHERE {
  ?a ?b ?c
}
```

is formatted to
```sparql
SELECT ?a # comment WHERE {
  ?a ?b ?c
}
```

So let's fix this really quickly, how hard can it be?  Right...?  

---

**Step 1**: Parse, does not change.  
The only difference is that comments can appear anywhere in the syntax tree:

![](img/FormattingComments.svg)

**Step 1.5**: Extract comments  
When collecting the edits, just ignore the comment-nodes.  

![](img/FormattingExtractComments.svg)

For each comment store:

- content
- indentation level
- is it a trailing comment or not
- the end position of the node it "attaches" to

The "attach" node is the first previous non-comment sibling or the parent.  
(Every comment has a parent.)

![](img/FormattingCommentAttach.svg)

Then do **Step 2**, **Step 3** and **Step 4** as before.  
But don't remove redundant edits in **Step 4**.

{{< notice note >}}

Since I compute a separation edit between each node, I can safely assume that each comment got "deleted"  
(except if it is the first or last child of the root but let's ignore this edge case).

{{< /notice >}}

**Step 5**: Merge comments  
Merge the extracted comments into the sequence of edits.

{{< notice note >}}

Since edits got consolidated in **Step 4**, I can assume that the edits left in the sequence are non-consecutive.
{{< /notice >}}

The location of the comment's "anchor" is either at the start of a separation edit:

![](img/FormattingInsertingCommentsCase1.svg)

Or contained in a merged edit:

![](img/FormattingInsertingCommentsCase2.svg)

In the second case, simply split the edit.  
This is safe, since it was merged in the first place.

Then, we edit the following textedit and remove the leading whitespace, except linebreaks.  
Then there are 3 cases:

![](img/FormattingEditTheEdit.svg)

If the edit is just whitespace, then replace it with a "linebreak"-edit.  
If the first non-whitespace character is not a linebreak, insert a "linebreak"-edit.  
If the first non-whitespace character is a linebreak, don't do anything.

---

Figuring out this all and implementing it properly took me about a week.

{{< notice warning >}}
  I think there is a simpler approach to handle comments in formatting.  
  Instead of *fiddling* in comment edits into the edit sequence,  
  it should be possible to handle them in the "**Step 2** - separation".  
  Simply by checking if there are comments between two nodes and changing the separator edit right then and there.  
  I will implement this approach in the future™.

{{< /notice >}}

#### Ideas for the future

- **Long lines**: should cause a line break

```sparql
SELECT ?v1 ?v3 ?v4 ?v5 ?v6 ?v7 ?v8 ?v9 ?v10
       ?v11 ?v12 ?v13 ?v14 ?v15 ?v16 ?v17
       ?v18 ?v19 ?v20 ?v21 ?v22 ?v23 ?v24
       ?v25 ?v26 ?v27 ?v28 ?v29 ?v30 ?v31
WHERE {}
```
- **Small nested queries**: should be compressed
```sparql
SELECT ?castle WHERE {
    osmrel:51701 ogc:contains ?castle .
    { { ?castle osmkey:historic "castle" }
      UNION
      { ?castle osmkey:historic "tower" . ?castle osmkey:castle_type "defensive" } }
    UNION
    { ?castle osmkey:historic "archaeological_site" . ?castle osmkey:site_type "fortification" }
    ?castle osmkey:name ?name .
    ?castle osmkey:ruins ?ruins .
    OPTIONAL { ?castle osmkey:historic ?class }
    OPTIONAL { ?castle osmkey:archaeological_site ?class }
}
```


### Hover

When the client sends a `textDocument/hover` request, the server responds with information about a given position in a text-document.

This is useful to give inexperienced users a documentation about how some constructs work:

 ![](img/examples/hover-filter.png)

#### Ideas for the future

**These are not implemented yet**

Display the structure of the query;
![](img/examples/hover-graph.png)
In a serious implementation, since these graph-patterns can get quite complex, I would use [mermaid.js](https://mermaid.js.org/) to generate diagrams. This would require a custom plugin in the editor to render these diagrams.

If the language server can query the knowledge graph (not implemented yet),
it could display additional information retrieved from the knowledge graph,
for example the label of a resource:
![](img/examples/hover-curie.png)
Or the incoming and outgoing edges:
![](img/examples/hover-detailed.png)

I'm sure there are many more ways to use "hover" to provide information to the user,
if you have a cool idea, [open an issue on github](https://github.com/IoannisNezis/Qlue-ls/issues) :)

### Diagnostics

A diagnostic is information provided by the language server about an issue in the source code.
They can be of different *severity*:
- **Error**: Critical issues; must be fixed (e.g., syntax errors, undefined variables).
- **Warning**: Potential problems; should be reviewed (e.g., deprecated functions, unused variables).
- **Info**: General insights; optional attention (e.g., code style improvements, documentation suggestions).
- **Hint**: Suggestions for enhancement (e.g., refactor opportunities, alternative approaches).

Diagnostics are either published by the server via a `textDocument/publishDiagnostics` notification,
or requested by the client via a `textDocument/diaglostic` request.

The diagnostics I implemented are:
- an error diagnostic for undeclared prefixes
- a warning diagnostic for unused prefixes
- an information diagnostic for uncompressed URI's

![](img/examples/diagnostics.png)

#### Ideas for the future

Highlight syntax errors:

![](img/examples/diagnostics-syntax.png)
This would require a more powerful parser that gives information about how the parse failed.

Highlight semantic errors, like selecting all variables in a `GROUP BY`:

![](img/examples/diagnostic-syntax.png)

Giving hints to make queries more concise:
![](img/examples/diagnostic-compress-triples.png)
![](img/examples/diagnostic-compress-obj.png)

### Code Actions

The `textDocument/codeAction` request is sent from the client to the server to request a set of commands for a given range in a textdocument. These commands can have arbitrary effects, but in most cases, they change the textdocument through textedits.

{{< notice note >}}
The change of the textdocument is always done by the client (editor).
The server provides the textedits, but its up to the client to apply them, since it "owns" the textdocument.
{{< /notice >}}

Often, code actions correspond to a diagnostic they resolve. Such code actions are called "quickfix".
The exemplary code action I implemented is "Shorten URI".
The idea is to shorten a URI into its compact ["Curie"](https://www.w3.org/TR/2010/NOTE-curie-20101216/) form.

| Before code action                       | After code action                       |
| ---------------------------------------- | --------------------------------------- |
| ![](img/examples/code-action-before.png) | ![](img/examples/code-action-after.png) |

This code action is powered by [curies.rs](https://github.com/biopragmatics/curies.rs).

### Completion Suggestions
The key feature for a SPARQL language server is, in my opinion, code-completion.
Here, the editor provides suggestions to the user.
In SPARQL, this is not just a convenience. Without smart completion suggestions, a user has to know the ontology by heart. Its also a massive efficiency boost for experienced users.

Here is a possible structure for completions:

#### 1. Static

The simplest kind of completion is just to provide static snippets of keywords or structures to the user.
For this, just the "location" of the cursor is relevant, the context of the content of the knowledge-graph does not influence the suggestions.

Here are a few examples i implemented:

|                                                |                                                |                                     |
| ---------------------------------------------- | ---------------------------------------------- | ----------------------------------- |
| ![](img/examples/cmp_select_pre.png)           | ![](img/examples/cmp_filter_pre.png)           | ![](img/examples/cmp_bind_pre.png)  |
| ![](img/examples/cmp_select_post.png)          | ![](img/examples/cmp_filter_post.png)          | ![](img/examples/cmp_bind_post.png) |

#### 2. Dynamic

Dynamic completions are more complex. They use all information available to provide context-based suggestions.

This again can be split into two categories:

#### 2.1. Offline

Here, the language server only uses "locally available" information:
That is: text in the editor and the data bundled with the language server (for example known prefixes).

A simple example for these are suggestions of already defined variables:
  
![](img/examples/cmp_variable.png)
This uses the parse tree to find all variables.
Currently, this is done very naively, as this also suggests variables when they are out of scope:

![](img/examples/cmp_variable_dumb.png)

#### 2.1. Online

Here, the language server uses data from a SPARQL-endpoint to provide completion suggestions.
{{< notice warning >}}
This is not implemented yet!
{{< /notice >}}

Here is an example from the Qlever-OSM-endpoint: <htps://qlever.cs.uni-freiburg.de/api/osm-planet/>.
[**O**pen**S**treet**M**ap](https://www.openstreetmap.org/) (OSM) is a publicly accessible project that collects geodata. Basically Google Maps, just without Google.
This data can be represented in an RDF knowledge graph and queried using SPARQL.
For example, this query returns every bicycle parking spot in [Freiburg](https://www.openstreetmap.org/relation/62768):
```sparql
PREFIX geo: <http://www.opengis.net/ont/geosparql#>
PREFIX osmkey: <https://www.openstreetmap.org/wiki/Key:>
PREFIX ogc: <http://www.opengis.net/rdf#>
PREFIX osmrel: <https://www.openstreetmap.org/relation/>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
SELECT ?bicycle_parking ?geometry WHERE {
  osmrel:62768 ogc:sfContains ?bicycle_parking .
  ?bicycle_parking osmkey:amenity "bicycle_parking" ;
                   geo:hasGeometry/geo:asWKT ?geometry .
}
```

Now, I want to know the same thing, just for Berlin.  
Unfortunately, I forgot that the relation ID of Berlin is `62422`...  
A good online-contextual-completion can help me:

|                            |                           |
| -------------------------- | ------------------------- |
| ![](img/examples/online-cmp-before.png) | ![](img/examples/online-cmp-after.png) |

# Using the language server

Ok, now let's talk about how to use the language server.
An editor needs to have a language server client and connect to our server.

I looked at three editors: neovim, vs-code, and a custom web-based-editor.

## Neovim

To get a language server running with neovim is very easy because it has a built-in language client and
neovim is built to be hacked.

### Installing the Program

First, `Qlue-ls` needs to be available as an executable binary on the system, neovim runs on.
You could just build the binary from source.
But to make it more convenient, I added the binary into two "repositories":

The Rust repository [crate.io](https://crates.io/crates/qlue-ls). To install from there, run:
```shell
cargo install qlue-ls
```
And the python repository [Pypi](https://pypi.org/project/qlue-ls/). To install from there, run:
```shell
pipx install qlue-ls
```
The python package is built with the help of [maturin](https://github.com/PyO3/maturin).

### Attaching

All you need is a `init.lua` file in your config directory with the following snippet:

```lua
vim.api.nvim_create_autocmd({ 'FileType' }, {
  desc = 'Connect to qlue-ls',
  pattern = { 'sparql' },
  callback = function()
    vim.lsp.start {
      name = 'qlue-ls',
      cmd = { 'qlue-ls', 'server' },
      root_dir = vim.fn.getcwd(),
      on_attach = function(client, bufnr)
        vim.keymap.set('n', '<leader>f', vim.lsp.buf.format, { buffer = bufnr, desc = 'LSP: ' .. '[F]ormat' })
      end,
    }
  end,
})
```

When you open a `sparql` file (a file with suffix `.rq`), this runs the command `qlue-ls server` and
connects the language client to this process.

## VS-Code

In VS-Code, there is no built-in language-client.
Instead, you have to create a vs-code-extension that acts as a language-client.
I will do that in the future™.

## The Browser

Okay, now the cherry on top:
Let's connect this thing to a web-based editor (an editor that runs in a browser).
"But wait!" you might say, "Browser is JavaScript land!".
And you would have been right, until 2017.

### WebAssembly (WASM)

[WebAssembly](https://webassembly.org/) is an open standard defined by the [World Wide Web Consortium](https://www.w3.org/).
It defines a bytecode to run programs within the browser.
All big browsers support it.

If your program can be converted (compiled) to this WebAssembly bytecode, it can execute in the browser.
![](img/WebAssembly-data-flow-architecture.png)[^8]

So, now, we need to write js-glue-code to call the Rust functions.  
Fortunately, [some strangers on the internet](https://github.com/rustwasm/team) already did that.
The project is called [wasm-pack](https://rustwasm.github.io/wasm-pack/).

It's very organic and simple, you just annotate the method or struct you want to "export" to WASM.

```rust
#[wasm_bindgen]
pub fn init_language_server(writer: web_sys::WritableStreamDefaultWriter) -> Server {
    wasm_logger::init(wasm_logger::Config::default());
    Server::new(move |message| send_message(&writer, message))
}
```

This glue code does a lot of stuff to bridge the gap between JavaScript and WASM.

```javascript
...
export function init_language_server(writer) {
    const ret = wasm.init_language_server(writer);
    return Server.__wrap(ret);
}
...
```

To actually run this in the browser, I needed to jump through a couple more hoops but I spare you the details.  
I packaged the result and uploaded it to [npm](https://www.npmjs.com/package/qlue-ls) - a JavaScript repository.
Now we can install the package using npm and access it from a JavaScript file:

```bash
npm install qlue-ls
```

```javascript
import { init_language_server } from "qlue-ls";
const server = init_language_server(...);
server.listen(...);
```

#### Tree-Sitter in WebAssembly

As stated earlier, tree-sitter creates a parser in C and I wrote my program in Rust.
It turns out compiling a Rust program that calls external C functions to WebAssembly creates something
called "**ABI-Incompatibilities**".
I wish I could tell you how I solved that, but to be honest, I don't want to talk about this experience, since it was extremely painful.

In short, I found a [project](https://github.com/shadaj/tree-sitter-c2rust) that ports the C code to Rust and all my issues disappeared.
This is the second reason for me to move away from tree-sitter.

### The Editor

Next, we need a web-based editor.
There are a couple options. Here are a few:

- [Monaco-Editor](https://microsoft.github.io/monaco-editor/)
- [CodeMirror](https://codemirror.net/)
- [ACE Editor](https://ace.c9.io/)

None of them have built-in language clients, but all of them have extensions that provides one.

- Monaco-Editor: [monaco-languageclient](https://github.com/TypeFox/monaco-languageclient)
- CodeMirror: [@shopify/codemirror-language-client](https://github.com/shopify/theme-tools), [codemirror-languageserver](https://github.com/FurqanSoftware/codemirror-languageserver), [codemirror-languageservice](https://github.com/remcohaszing/codemirror-languageservice)
- ACE Editor: [ace-linters](https://github.com/mkslanc/ace-linters)

But, to be fair, the ones for CodeMirror don't really look stable.

I decided to go with Monaco for a couple reasons:

- It's very mature (It's the same editor that drives vs-code)
- The language-client extension looks very active.
- I hope to get a synergy effect since both LSP and Monaco are from Microsoft

I will eventually also try ACE.

To get Monaco running with the language client, I had to jump through a few hoops again (actually a lot),
but I again spare you with the details.

### TextMate

A good editor needs syntax highlighting. (The thing that makes the colors.)

| without syntax highlighting              | with syntax highlighting              |
| ---------------------------------------- | ------------------------------------- |
| ![](img/examples/highlighting_off.png)   | ![](img/examples/highlighting_on.png) |

For Monaco-Editor, there are 2 options:

- [Monarch](https://microsoft.github.io/monaco-editor/monarch.html): built specifically for Monaco, well suited for simple languages
- [TextMate Grammar](https://macromates.com/manual/en/language_grammars): feature of the [TextMate](https://macromates.com/) Editor, widely used standard in text editors 

I went with TextMate Grammar.
It's more complex but also more powerful and can be used with other editors.

TextMate Grammars use the [Oniguruma](https://github.com/kkos/oniguruma) regex engine.
Basically, you write patters that identify the tokens you want to highlight.

Here is a simple example:

```json
 {
	"scopeName": "source.sparql",
	"fileTypes": ["sparql"],
	"foldingStartMarker": "\\{\\s*$",
	"foldingStopMarker": "^\\s*\\}",
	"patterns": [
		{
			"name": "keyword.control.sparql",
			"match": "\\b(SELECT|WHERE|FILTER|)\\b"
		},
		{
			"name": "string.quoted.double.sparql",
			"begin": "\"",
			"end": "\"",
			"patterns": [
				{
					"name": "constant.character.escape.sparql",
					"match": "\\\\."
				}
			]
		}
	]
}
```

### Plugging everything together

Now, this actually falls under "hoops I needed to jump through".
But I think its quite aesthetic.

Here are the pieces we have so far:

The DOM-Element, the Monoco-Editor-Worker, the TextMate-Worker, the Language-Server.
With Worker, I mean [Web Worker](https://de.wikipedia.org/wiki/Web_Worker). Web-Workers allow JavaScript code to be executed separately from the main-thread.

All that's left is a Worker that forwards the language-server WASM component.

Here is the architecture:

![](img/setup.png)

# How does Qlue-ls compare against other software?

Now we have a language-server we can use in various editors that have limited capabilities.
How does this compare against other software that's out there?

Here is what I found:

| Tool                                                                                | Description                                                                                                               | Platform     | Maintainer                                                             | FOSS          |
| ----------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------------------------------------------------------------- | ------------- |
| [sparql-langauge-server](https://github.com/stardog-union/stardog-language-servers) | SPARQL language-server build in TypeScript                                                                                | web & native | [Stardog](https://www.stardog.com/)                                   | ✅  Apache-2.0 |
| [RDF and SPARQL plugin](https://plugins.jetbrains.com/plugin/13838-rdf-and-sparql)  | A RDF and SPARQL plugin for JetBrains IDEs                                                                                | native       | [Sharedvocabs Ltd](https://plugins.jetbrains.com/vendor/sharedvocabs) | ❌ 5€/month    |
| [Qlever-UI](https://github.com/ad-freiburg/qlever-ui)                               | A custom [codemirror](https://codemirror.net/) editor for the [Qlever](https://github.com/ad-freiburg/QLever) triplestore | web          | [ad-freiburg](https://ad.informatik.uni-freiburg.de/)                 | ✅  Apache-2.0 |
| [YASGUI](https://github.com/TriplyDB/Yasgui)                                        | A lightweight, web-based SPARQL editor                                                                                    | web          | [Triply](https://triply.cc/en-US)                                     | ✅  MIT        |
| [sparql-formatter](https://github.com/sparqling/sparql-formatter/tree/main)         | A SPARQL formatter built in JavaScript                                                                                    | web          | [SPARQLing](https://github.com/sparqling)                             | ?             |

Let's look at these tools' comparison concerning the aspects discussed in this article. (**personal opinion**)

| Feature                      | Qlue-ls | Stardog's SPARQL-ls | JetBrains Plugin | Qlever-UI              | YASGUI |
| ---------------------------- | ------- | ------------------- | ---------------- | ---------------------- | ------ |
| Formatting                   | ⭐⭐⭐     | ❌                   | ⭐⭐⭐              | ⭐⭐⭐<br>(using Qlue-ls) | ⭐⭐     |
| Hover (Offline)              | ⭐       | ⭐                   | ⭐⭐               | ❌                      | ❌      |
| Hover (Online)               | ❌       | ❌                   | ❌                | ⭐⭐                     | ❌      |
| Diagnostics                  | ⭐⭐      | ⭐                   | ⭐⭐               | ❌                      | ⭐      |
| Code-actions                 | ⭐⭐      | ❌                   | ⭐⭐               | ❌                      | ❌      |
| Completion - offline-static  | ⭐⭐      | ⭐                   | ⭐⭐⭐              | ⭐⭐                     | ❌      |
| Completion - offline-dynamic | ⭐       | ⭐                   | ⭐                | ⭐⭐                     | ⭐      |
| Completion - online-dynamic  | ❌       | ❌                   | ❌                | ⭐⭐                     | ⭐      |

| Symbol | Meaning         |
| ------ | --------------- |
| ⭐⭐⭐    | nearly perfect  |
| ⭐⭐     | could be better |
| ⭐      | barely working  |
| ❌      | not implemented |

{{< notice warning >}}
These observations came from a brief inspection. It's possible that they are better than I think they are!
For example, `YASGUI` also supports custom queries for completion. I did not have the time to test this properly!
I know that `Qlever-UI` also uses custom queries, so the comparison between the two may be unfair.
{{< /notice >}}

## Qlue-ls vs sparql-formatter

Since [sparql-formatter](https://github.com/sparqling/sparql-formatter/tree/main) is "just" a formatter, it's not really fair to compare it like the other tools.
So let's do a direct comparison:

I built a scraper that collected all 381 example queries from the [Wikidata Query Service](https://query.wikidata.org/).
Then I formatted the query with `Qlue-ls` and `sparql-formatter` and compared the two results using diff.
Here are the differences:

### Core differences

#### Error resilience

`sparql-formatter` expects correct queries, while `Qlue-ls` also works with incomplete queries.
The quality of the result depends on how the parser handles the error.
#### Concrete vs Abstract
`Qlue-ls` uses a Concrete Syntax Tree (*CST*) and `sparql-formatter` uses an Abstract Syntax Tree (*AST*).

While a *CST* represent the **complete** syntactic structure of the input (including parentheses, punctuation ...),
an *AST* is an abstracted representation. It omits syntactic details and only keeps the essential elements.

For example: for an input like `(22 * (3 - 1) - 2)`
the trees would be:

| CST          | AST          |
| ------------ | ------------ |
| ![](img/cst.png) | ![](img/ast.png) |

While you can use both for formatting, a *CST* guarantees that no token gets lost or is added.
For example, `sparql-formatter` adds a "." here:

| before                     | after                     |
| -------------------------- | ------------------------- |
| ![](img/examples/format_ast_before.png) | ![](img/examples/format_ast_after.png) |

### Errors

Let's look at critical errors the formatters do (or don't do)

#### Errors in sparql-formatter

#### Strange Brackets in Expression lists

`sparql-formatter` throws an error for this query:
![](img/examples/format-sq-error1.png)
```
SyntaxError at line:5(col:11-12)
Expected: != && * + - / < <= = > >= IN NOT ||
--
          )

```
The brackets are strange but no reason to throw an error.

#### Encapsulated Aggregate
`sparql-formatter` throws an error for encapsulated aggregates:

![](img/examples/format_error_3.png)

```
file:///usr/lib/node_modules/sparql-formatter/src/formatter.js:811
  if (variable.varType === '$') {
               ^

TypeError: Cannot read properties of undefined (reading 'varType')
```
#### Errors in Qlue-ls formatter

None I know of.

### Opinion-based differences

#### Where-Clause newline

| Qlue-ls                   | sparql-formatter          |
| ------------------------- | ------------------------- |
| ![](img/examples/format_diff_1_ql.png) | ![](img/examples/format_diff_1_sf.png) |

#### Union-Clause newline

| Qlue-ls                   | sparql-formatter          |
| ------------------------- | ------------------------- |
| ![](img/examples/format_diff_2_ql.png) | ![](img/examples/format_diff_2_sf.png) |

#### Capitalize Keywords

`sparql-formatter` capitalizes some, but not all keywords.
`Qlue-ls` capitalizes all.

| Qlue-ls                   | sparql-formatter          |
| ------------------------- | ------------------------- |
| ![](img/examples/format_diff_3_ql.png) | ![](img/examples/format_diff_3_sf.png) |

#### Line length based line breaks

sparql-formatter will add a linebreak if the line get to long.

| Qlue-ls                    | sparql-formatter          |
| -------------------------- | ------------------------- |
| ![](img/examples/format_error_3_ql.png) | ![](img/examples/format_diff_4_sf.png) |
### Summary

`sparql-formatter` has line-length-based line breaks, but has a few runtime errors, adds tokens (non-breaking) and has inconsistent capitalization.

`Qlue-ls` is error-resilient and has no known runtime errors.

Overall, the two create surprisingly similar outputs.
I opened up an issue for the bugs I found, so maybe, in the future, we will produce the same output (beside the `CST`/`AST` stuff).

I also wanted to do a performance comparison.
But creating arbitrarily long random SPARQL queries is a side quest I don't have the time for.

# Future work

`Qlue-ls` is still in the alpha phase right now (07-01-2025).
It was my first Rust project and first contact with linked-data.
I'm proud of my work, but I'm sure there is a lot to improve!

Aside the code quality, efficiency and so on - here is the roadmap for this project.

## Stronger Parser

As stated earlier, tree-sitter was nice to get going fast, but it's not the right solution for this problem.
I need a parser that **deterministically** gives me the exact "location" within the parse-tree for any position in the editor.
This could be achieved with a resilient "LL(1)"-parser.
There is an [article](https://matklad.github.io/2023/05/21/resilient-ll-parsing-tutorial.html) by [Alex Kladov](https://github.com/matklad), the creator of [rust-analyzer](https://github.com/rust-lang/rust-analyzer) (**the** rust language server) that goes into detail regarding this topic. This will be the topic of my Bachelor Thesis.

## Query SPARQL endpoint

Currently, `Qlue-ls` it not firing any queries against a SPARQL-endpoint.
This will open a new world of cool features and make this language server much more useful.

There are two reasons I did not implement this yet:
1. I currently ship to WASM **and** x86 targets, which makes this a bit more complex.
2. This opens the box of **async** and I heard that that is another level of complexity in Rust.

## Enhance existing features

Formatting should try to maintain a maximal line length.
It could also try to keep things more concise.

Diagnostics and code-actions can be extended majorly.

I did not implement this since it's mostly repetitive extra work that does not bring any new insights.
Also, since all existing features depend on the parse-tree, I want to build a stronger parser first!
# Acknowledgements

- [TJ DeVries](https://github.com/sponsors/tjdevries) for awesome LSP tutorials:
	- [Building a Language-Server from scratch](https://www.youtube.com/watch?v=YsdlcQoHqPY)
	- [LSP in neovim](https://www.youtube.com/watch?v=bTWWFQZqzyI)
- [Gordian Dziwis](https://github.com/GordianDziwis/tree-sitter-sparql) for doing the heavy lifting
- [guyutongxue](https://github.com/Guyutongxue/clangd-in-browser) for showing how to run clangd in the browser

[^1]: https://tree-sitter.github.io/tree-sitter/
[^2]: https://microsoft.github.io/language-server-protocol/overviews/lsp/overview/
[^3]: https://www.zeit.de/video/2024-11/6364737354112/neuseeland-abgeordnete-protestieren-mit-haka-tanz-im-neuseelaendischen-parlament
[^4]: notifications are messages without a `id`, since they do not expect a response.
[^5]: In rust packages are called crates (you know... because cargo ships crates...)
[^6]: https://www.youtube.com/watch?v=wLg04uu2j2o
[^7]: https://code.visualstudio.com/api/language-extensions/language-server-extension-guide
[^8]:https://www.researchgate.net/figure/WebAssembly-data-flow-architecture_fig1_373229823
