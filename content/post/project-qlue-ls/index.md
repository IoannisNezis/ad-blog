---
title: "Qlue-ls a SPARQL language server"
date: 2025-01-12T12:43:04+02:00
author: "Ioannis Nezis"
authorAvatar: "img/profile.png"
tags: ["SPARQL", "lsp", "wasm", "rust"]
categories: []
image: "img/cover.png"
---

Modern developer environments are way more capable than simple Text editors.
They provide domain-specific tools to improve the user experience.
They give hints, suggest changes or completions and more.  
In this article we will take a look behind the curtains and build language support for *SPARQL*,
a query language for knowledge graphs.

<!-- more -->

You can find the source code in my [GitHub repository](https://github.com/IoannisNezis/sparql-language-server).  
And a life demo on [qlue-ls.com](https://qlue-ls.com/).


# TL;DR

I build a [sparql-language-server](https://github.com/IoannisNezis/Qlue-ls) from scratch in [Rust](https://www.rust-lang.org/), powered by [tree-sitter](https://tree-sitter.github.io/tree-sitter/).
To showcase the Language server I build a [web editor](https://sparql.nezis.de/) using [Monaco](https://microsoft.github.io/monaco-editor/).
To run the language server within the browser, I used [WebAssembly](https://webassembly.org/)

# Content



- [Motivation](#motivation)
- [Goal](#goal)
- [The Language Server Protocol ](#the-language-server-protocol)
    - [JSON-RPC](#json-rpc)
    - [Document synchronization](#document-synchronization)
    - [Capabilities](#capabilities)
- [Implementation](#implementation)
    - [speakingJSON-RPC](#speaking-json-rpc)
    - [Parser: the Engine under the hood](#parser:-the-engine-under-the-hood)
        - [Tree-sitter](#tree-sitter)
    - [Implemented Capabilities](#implemented-capabilities)
        - [Formatting](#formatting)
        - [Hover](#hover)
        - [Diagnostics](#diagnostics)
        - [Code Actions](#code-actions)
        - [Completion Suggestions](#completion-suggestions)
            - [ Static](#1-static)
            - [ Dynamic](#2-dynamic)
            - [ Offline](#21-offline)
            - [ Online](#21-online)
- [Using the Language server](#using-the-language-server)
    - [Neovim](#neovim)
        - [Installing the Programm](#installing-the-programm)
        - [Attaching](#attaching)
    - [VS-code](#vs-code)
    - [The Browser](#the-browser)
        - [WebAssembly](#webassembly)
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

>[!todo]
> idea: editor keeps state of you document, language server the state of  your code,
> lsp enables the exange between those two informations

The problem of providing language support to developers is very old.
In the past domain-specific development environments where very common.

| Domain    | development environment |
| --------- | ----------------------- |
| Java      | Eclipse                 |
| Microsoft | Visual Studio           |
| C/C++     | Turbo C++               |
| python    | PyCharm                 |
| R         | RStudio                 |
| LaTeX     | TeXworks or Overleave   |

These Programs contain source-code editors, but also provide a suit of integrated tool's that support the development process of their respective domains. That's why they are also referred to IDE's (**I**ntegrated **D**evelopment **E**nvironment).
While these development environments still dominate, modern development environments seem to go a different direction.

Some of the new kids on the block are: [neovim](https://neovim.io/), [vscode](https://code.visualstudio.com/) or [sublime text](https://www.sublimetext.com/).
They all are **general purpose** code editors that have a open-source plugin ecosystem and allow for a personalized customization. A core maintainer of Neovim, [TJ DeVries](https://github.com/tjdevries), calls them PDE's (**P**ersonalized **D**evelopment **E**nvironment), although i don't think it caught on yet.

Long story short: Language support in these PDE's is not build in, but provided via a extension.
This is made possible by a Protocol published by Microsoft in 2016: The **L**anguage **S**erver **P**rotocol (LSP).
It enables the Editor (LSP-Client) and the Language support program (LSP-Server or Language Server) to be separated into to two independent components.

![](img/language-server-sequence.png)
[^2]

A key advantage of this architecture is that this features for language support has to be only written once, and not for every development tool over and over again.

![](img/lsp-languages-editors.png)[^8]
# Goal

My goal is to create a Language Server for [SPARQL](https://www.w3.org/TR/sparql11-query/#rQueryUnit).
The language server should be able to **format** queries, give **diagnostic** reports and **suggest completions**.
To work in the [Qlever-UI](https://qlever.cs.uni-freiburg.de/) the Language Server should be accessible from an editor that runs in the browser.
# The Language Server Protocol

Let's talk briefly about the Protocol.
It's build on top of [JSON-RPC](https://www.jsonrpc.org/specification), a `json` based protocol that enables, as the name suggests, inter-process communication.
So the development tool (in our case the editor) and the language server run in two separate processes and communicate asynchronously via **JSON-RPC**.

## JSON-RPC

I will give you just a brief introduction, if you want to know more read the [specification](https://www.jsonrpc.org/specification#id1).

A normal JSON-RPC request has an `id`, `method` and `params` field.
The `method` is the name of the invoked operation, and `params` field contains the optional parameters for this invoked operation.

A normal JSON-RPC response has an `id` and `result` field.
The `id` has to be the same as the `id` of the corresponding request. This enables async communication.
The `params` contain the result of the operation, if there is one.

>[!example]
> Request: `{"jsonrpc": "2.0", "method": "add", "params": [21, 21], "id": 1}`
> Response: `{"jsonrpc": "2.0", "result": 42, "id": 1}`

There are also notifications[^4] and error responses, but we will omit them for now.

## Document synchronization

For the Language server to do anything it needs to know what the user is doing.

The Specification defines 3 [Document Synchronization](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_synchronization) methods for this:
 - [`textDocument/didOpen`](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_didOpen)
 - [`textDocument/didChange`](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_didChange)
 - [`textDocument/didClose`](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_didClose)
which are mandatory to implement (for clients).

The names are pretty self-explanatory. When ever a document is opened, changed or closed the client sends this information to the server. The [textDocument/didChange](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_didChange) notification[^4] supports full and incremental synchronization. The server and client negotiate this during [initialization](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#initialize).

Incremental changes were more diffucult than expected.
This is mainly because of the translation between different encodings.
Editors give the position in the textdocument (row, column) based on a utf-16 string representation.
While the the chars them self are encoded in utf-8.
Ofcourse utf-8 is a variable length encoding, so different characters can have different byte-sizes.

This was a bit confusing to get right.

![](img/encodings.png)


Through these messages the language server has a "mirrored", synchronization version of the editor state.

>[!example]
> Here is a example for a incremental **textDocument/didChange** notification.
> ```json
> {
>	"params": {
>		"contentChanges": [
>			{
>				"rangeLength": 6, //deprecated
>				"text": "42",
>				"range": {
>					"end": {
>						"line": 3,
>						"character": 22
>					},
>					"start": {
>						"line": 3,
>						"character": 16
>					}
>				}
>			}
>		],
>		"textDocument": {
>			"uri": "file:\/\/\/home\/ianni\/code\/sparql-language-server\/example.sparql",
>			"version": 227
>		}
>	},
>	"jsonrpc": "2.0",
>	"method": "textDocument\/didChange"
>}
>```

## Capabilities

When Initialization and synchronization work, the real fun begins.
Now we can implement complex language features and provide actual smarts to the editor.
As long as both the client and server support the capability.

Here is an **incomplete** list of Language Feature capabilities that made it into the Specification

| capability                                                                                                                                         | effect                                                        | State of implementation |
| -------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------- | ----------------------- |
| [Go to Declaration](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_declaration)          | Jumps to the declaration of a symbol                          | Not planned             |
| [Go to Definition](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_definition)            | Jumps to the definition of a symbol                           | Not planned             |
| [Document Highlight](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_documentHighlight)   | Highlight  all references to a symbol                         | Planned                 |
| [Document Link]()                                                                                                                                  | Handle Links in a Document                                    | Planned                 |
| [Hover](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_hover)                            | Show Information about the hovered symbol                     | In progess              |
| [Folding Range](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_hover)                    | Indentify  foladable ranges in the document.                  | Not planned             |
| [Document Symbols](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_documentSymbol)        | Identify all symbols in a document                            | Not planned             |
| [Inline Value](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_inlineValue)               | Show Values in the editor                                     | Not Planned             |
| [Completion Proposals](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_completion)        | Give Completion Proposals to the user                         | In progress             |
| [Publish Diagnostics](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_publishDiagnostics) | Send Information like Hints, Warnings or Errors to the editor | In progress             |
| [Pull Diagnostics](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_pullDiagnostics)       | Request Information like Hints, Warningso Errors              | In progress             |
| [Code Action](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_codeAction)                 | Suggest changes or fixes like refactoring                     | In progress             |
| [Formatting](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_formatting)                  | Format the whole document                                     | Done                    |
| [Range Formatting](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_rangeFormatting)       | Format the provided range in a document                       | Not Planned             |
| [Rename](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_rangeFormatting)                 | Rename a symbol                                               | Planned                 |

At the moment (05.12.2024) only the formatting feature is really completed.
The rest is in an experimental state and needs a stronger engine to work properly.

# Implementation

Let's talk about what I actually did.
I choose to use [Rust](https://www.rust-lang.org/) for this project since its fancy in I like shiny things.
Rust is the most admired programming language in the [Stack overflow developer survey 2024](https://survey.stackoverflow.co/2024/technology#2-programming-scripting-and-markup-languages), and I was curious why. After this project I can confirm that Rust is a very cool language, but the leading curve is quiet steep.
Especially the error handling, increddibly smart compiler, functional approach and rich ecosystem enable a smooth developing experience. Besides a steep learning curve i also feel like its harder to get stuff done quickly, compared to python, due to the very strict compiler. But the resulting code is a lot more robust.

Okay first things first, we need to speak **JSON-RPC**.
Then we need to implement some tool to analyse SPARQL queries.
When we got the analysis tool running we can use it to provide some language features.

Here is the module structure of my crate[^5]:

![](img/examples/code structure.canvas|code structu)

## speaking JSON-RPC

Assume we set up the Editor (client) to connect to our language server.
It will send an utf8-byte stream over the configured channel (stdio), and we need to interpret and respond to this byte steam.

The first message will look something like this:

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

For each type of message I build a set of corresponding structs:

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
Note that in Rust the notion of "Inheritance" does not exist. It uses "traits" to define shared behavior.
That's no issue here, with `#[serde(flatten)]` we can inline data from a struct into a parent struct and achieve a similar effect.
Another issue was the naming convention.
I JSON-RPC the fields are written in camelCase, but Rust uses snake_case.
Serde also offers a solution for that:  the `#[serde(rename_all = "camelCase")]` annotation, to convert this behind the scenes.

This is basically how I read and write messages.
I defined structs for the basic [lifecycle massages](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#initialize):

| message                                                                                                               | sender | type         | effect                                         |
| --------------------------------------------------------------------------------------------------------------------- | ------ | ------------ | ---------------------------------------------- |
| [initialize](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#initialize)  | client | request      | initialize connection, comunicate capabilities |
| [initalized](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#initialized) | client | notification | signal reception of initialize response        |
| [shutdown](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#shutdown)      | client | request      | shutdown server, dont exit                     |
| [exit](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#exit)              | client | notification | exit the server process                        |

Then I defined the basic structs for  [document sznchronization](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_synchronization)

| message                                                                                                                                      | sender | type         | effect                              |
| -------------------------------------------------------------------------------------------------------------------------------------------- | ------ | ------------ | ----------------------------------- |
| [textDocument/didOpen](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_didOpen)     | client | notification | signal newly opened text document   |
| [textDocument/didChange](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_didChange) | client | notification | signal changes to a text document   |
| [textDocument/didClose](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textDocument_didClose)   | client | notification | signal that text document is closed |

With these messages defined, we can open and close a connection to a client and keep a synchronized state of the clients documents.

To run this native i also needed to listen to stdin and parse the [Header](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#headerPart) from raw bytes but i spare you that experience.

## Parser: the Engine under the hood

Okay now we want to build some tools to analyze the given text,
and provide some smarts.

First we need to "understand" the given text.
Understanding arbitrary text is quiet the challenge, and we only recently made some advancements in that area.
Luckily all SPARQL queries follow a strict structure, called a grammar, which is defined in its [specification](https://www.w3.org/TR/sparql11-query/#rQuery).
A grammar is basically a set of production rules that map nonterminal-symbols to other nonterminal-symbols or terminal-symbols. Every valid SPARQL query can be produced by applying those rules until only terminal symbols are left.
lFor some grammars we can build a program that reconstructs which production rules got used to produce a given string. Such a program is called **parser**. The result of a parser, the rules that got used, is called **syntax tree**.

Here is a example:

**Query**:
```sparql
SELECT * WHERE {
  ?s ?p ?p
}
```

**Syntax tree:**
```lisp
(unit ; [0, 0] - [3, 0]
  (SelectQuery ; [0, 0] - [2, 1]
    (SelectClause ; [0, 0] - [0, 8]
      "SELECT" ; [0, 0] - [0, 6]
      "*") ; [0, 7] - [0, 8]
    (WhereClause ; [0, 9] - [2, 1]
      "WHERE" ; [0, 9] - [0, 14]
      (GroupGraphPattern ; [0, 15] - [2, 1]
        "{" ; [0, 15] - [0, 16]
        (GroupGraphPatternSub ; [1, 2] - [1, 10]
          (TriplesBlock ; [1, 2] - [1, 10]
            (TriplesSameSubjectPath ; [1, 2] - [1, 10]
              (VAR) ; [1, 2] - [1, 4]
              (PropertyListPathNotEmpty ; [1, 5] - [1, 10]
                (VAR) ; [1, 5] - [1, 7]
                (ObjectList ; [1, 8] - [1, 10]
                  (VAR)))))) ; [1, 8] - [1, 10]
        "}")))) ; [2, 0] - [2, 1]
```

### Tree-sitter

Usually parses get to parse complete strings that are valid.
When you give them invalid input they react not so happy (symbolic image below)

![](img/resiliant-parsers.png)
[^3]

In our use-case the text we handle is incomplete most of the time.
So we need a parser that is resilient to errors.

The big-boy-language-servers like [rust-analyser](https://github.com/rust-lang/rust-analyzer) use customized [resilient LL Parsers](https://matklad.github.io/2023/05/21/resilient-ll-parsing-tutorial.html).
But since i wanted to get on the road quick I choose [tree-sitter](https://tree-sitter.github.io/tree-sitter/) (for now... ;)).
Tree-sitter, at its core, is a parser generator. It generates error resilient parsers.
Its most commonly used in editors like [neovim](https://neovim.io/) or emacs[^6] to highlight text.

Parser generators take a grammar and generate a fully functioning parser.
I found a [sparql-tree-sitter](https://github.com/GordianDziwis/tree-sitter-sparql) grammar by [GordianDziwis](https://github.com/GordianDziwis).
It had a few minor issues and was a bit dusty.
So i [forked](https://github.com/GordianDziwis) it and it and changed a few things to work better with Rust.

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
For some C-reasons I won't get into right now, tree-sitter can generate rust-bindings that allow us to call the c-functions from our rust-program (something I will regret later).
It provides some functions to parse and navigate the resulting concrete-syntax-tree.

>[!example]
>Here is a example from the format function:
>```rust
>   let mut parser = Parser::new();
>   match parser.set_language(&tree_sitter_sparql::LANGUAGE.into()) {
>    Ok(()) => {
>        let tree = parser
>            .parse(text.as_bytes(), None)
>            .expect("could not parse");
>        let formatted_text =
>            format_helper(&text, &mut tree.walk(), 0, "  ", "", &format_settings);
>        return formatted_text;
>    }
>    Err(_) => panic!("Could not setup parser"),
>}
>```

A cool feature of tree-sitter is [queries](https://tree-sitter.github.io/tree-sitter/using-parsers#pattern-matching-with-queries).
A query is build out of one or more patterns. Each pattern is a [S-expression](https://en.wikipedia.org/wiki/S-expression).
Tree-sitter can match these patterns against the syntax tree and return all matches.

>[!example]-
>Let's say we want to find all *triples* in a group-graph-pattern that have a predicate that match "pre:p1":
>The query could look like:
>```sparql
>SELECT * WHERE {
>    ?s pre:p2 "object1" .
>    ?s2 pre:p1 "object2"
>}
>```
>My Tree-sitter parser would generate this parse-tree:
>```lsip
>(unit
>  (SelectQuery
>    (SelectClause)
>    (WhereClause
>      (GroupGraphPattern
>        (GroupGraphPatternSub
>          (TriplesBlock
>            (TriplesSameSubjectPath
>              subject: (VAR)
>              (PropertyListPathNotEmpty
>                predicate: (Path
>                  (PathSequence
>                    (PathEltOrInverse
>                      (PathElt
>                        (PathPrimary
>                          (PrefixedName
>                            (PNAME_NS
>                              (PN_PREFIX))
>                            (PN_LOCAL)))))))
>                (ObjectList
>                  object: (RdfLiteral
>                    value: (String
>                      (STRING_LITERAL))))))
>            (TriplesSameSubjectPath
>              subject: (VAR)
>              (PropertyListPathNotEmpty
>                predicate: (Path
>                  (PathSequence
>                    (PathEltOrInverse
>                      (PathElt
>                        (PathPrimary
>                          (PrefixedName
>                            (PNAME_NS
>                              (PN_PREFIX))
>                            (PN_LOCAL)))))))
>                (ObjectList
>                  object: (RdfLiteral
>                    value: (String
>                      (STRING_LITERAL))))))))))))
>```
>
>And here is the query:
>```lisp
>(TriplesSameSubjectPath
>  (VAR)
>  (PropertyListPathNotEmpty
>    (Path) @path (#eq? @path "pre:p1"))) @triple
>```
>`@path` and `@triple` are captures and store the nodes that match the pattern.

#### How resilient is tree-sitter?

Very, resilient.
Tree-sitter recovers from basically anything and conserves information from very incomplete input.
So the question is not how resilient it is, the problem is how good the quality of the information it conserves is.

Tree-sitter produces GLR parsers.
It explores, non-deteministically, many different possible LR (bottom-up) parses.
Then it chooses the "the best one".
In the words of [Alex Kladov](https://github.com/matklad) the creator of [rust-anayzer](https://github.com/rust-lang/rust-analyzer) this leads to the following behavior:
>(...) Tree-sitter \[can\] recognize many complete valid small fragments of a tree, but it might have trouble
> assembling them into incomplete larger fragments.

This is very useful for the use-case of syntax highlighting. You want to highlight as many tokens as possible, the larger context of in which they appear is often not so important.

In our use-case however, it's the other way around.

Let's look at a few examples.

## Implemented Capabilities

Okay, now that you may have an idea of the fundamental mechanisms.
Let's talk about the features, besides the lifecycle and synchronization, that I implemented.

>[!warning]
>The implemented features are just a **proof of concept**!
>There are many many features that could  and should be added.
>But that takes a lot of time to get right.
>And frankly also requires a stronger parser.
>This should just give you a idea of whats possible.

### Formatting

When the client sends a `textDocument/formatting` request, the server returns a list of [`TextEdit`](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/#textEdit)'s.
A `TextEdit` is a incremental change, composed of a Range and a String.
The Client then applies the changes and thus formats the document.

I implemented this with a post-order-[traversal](https://en.wikipedia.org/wiki/Tree_traversal) of the parse-tree provided by tree-sitter.

![](img/postorder-traversal.png)
[^7]

Each step the "Type" of the node is matched and handled with a strategy.
For example "sparate each child by a new line" or "capitalize the text".
That is done recursivly.

Here are a few examples of formatted queries.  
If you want get a more detailed picture, check out the [formatting tests](https://github.com/IoannisNezis/Qlue-ls/blob/main/src/server/message_handler/formatting/tests.rs).

- 
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
-
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
- 
    ```sparql
    SELECT * WHERE {
        ?a ?b ",,," .
        FILTER (1 IN (1, 2, 3))
    }
    ```
-
    ```sparql
    SELECT * WHERE {
        ?a <iri>/^a/(!<>?)+ | (<iri> | ^a | a) ?b .
    }
    ```

I added some optional formatting, like aligning prefix declarations and predicates.
Here is a example for a query with every non default option:

-
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
```

#### Ideas for the future

There are three things that the formatter does not handle well:

- **Comments**: should be possible to place at the end of a line:
    ```sparql
    SELECT * WHERE {
        ?s ?p ?o # EOL comment
    }
    ```

- **Long lines**: should cause a linebreak
    ```sparql
    SELECT ?v1 ?v3 ?v4 ?v5 ?v6 ?v7 ?v8 ?v9 ?v10
           ?v11 ?v12 ?v13 ?v14 ?v15 ?v16 ?v17
           ?v18 ?v19 ?v20 ?v21 ?v22 ?v23 ?v24
           ?v25 ?v26 ?v27 ?v28 ?v29 ?v30 ?v31
    WHERE {}
    ```
- **small nested queries**: should be compressed
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

When the client sends a `textDocument/hover` to request, the server responds with information about a given position in a text-document.

This is usefull to give unexperienced users documentation about how some constructs work:

 ![](img/examples/hover-filter.png)

#### Ideas for the future

Here are a few ideas i have to extend this feature.
**These are not implemented yet**

Display the structure of the query;
![](img/examples/hover-graph.png)
In a serious implementation, since these graph-patterns can get quiet complex, i would use [mermaid.js](https://mermaid.js.org/) to generate diagramms. This would require a custom plugin in the editor to render these diagramms.

If the language server can query the knwledge graph autonomously (not  implemented yet), it could display additional information retrieved from the knowledge graph.
For example the label of a resource:
![](img/examples/hover-curie.png)
Or the incoming and outgoing edges:
![](img/examples/hover-detailed.png)

Im sure there are many more ways to use "hover"  to provide information to the user,
if you have a cool idea, [open a issue on github](https://github.com/IoannisNezis/Qlue-ls/issues) :)

### Diagnostics

A diagnostic is information provided by the language server about a issue in the source code.
They can be be of different *severity*:
- **Error**: Critical issues; must be fixed (e.g., syntax errors, undefined variables).
- **Warning**: Potential problems; should be reviewed (e.g., deprecated functions, unused variables).
- **Info**: General insights; optional attention (e.g., code style improvements, documentation suggestions).
- **Hint**: Suggestions for enhancement (e.g., refactor opportunities, alternative approaches).

Diagnostics are either published by the server via a `textDocument/publishDiagnostics` notification,
or requested by the client via a `textDocument/diaglostic` request.

The diagnostics I implemented are:
- a error diagnostic for undeclared prefixes
- a warning diagnostic for unused prefixes
- a information diagnostic for uncompressed uri's

![](img/examples/diagnostics.png)

#### Ideas for the future

Hightlight syntax errors:

![](img/examples/diagnostics-syntax.png)
This would require a more powerfull parser, that gives information about how the parse failed.

Hightlight sematic errors, like selecting all variables in a `GROUP BY`:

![](img/examples/diagnostic-syntax.png)

Giving hints to make queries more concise:
![](img/examples/diagnostic-compress-triples.png)
![](img/examples/diagnostic-compress-obj.png)

### Code Actions

The `textDocument/codeAction` request is send from the client to the server to request a set of commands for a given range in a textdocument. These commands can have arbitrary effect, but in most cases change the textdocument through text-edits.

>[!note]
>The change of the textdocument is always done by the client (editor).
>The server provides the text-edits but its up to the client to apply them since it "owns" the textdocument.

Often code-actions corispond to a diagnostic they resolve. Such code-actions are called "quickfix".
The exemplary code action i implemented is "Shorten URI".
The idea is to shorten a uri into its compact ["Curie"](https://www.w3.org/TR/2010/NOTE-curie-20101216/) form.

| Before code action                       | After code action                       |
| ---------------------------------------- | --------------------------------------- |
| ![](img/examples/code-action-before.png) | ![](img/examples/code-action-after.png) |

This code-action is powered by [curies.rs](https://github.com/biopragmatics/curies.rs).

### Completion Suggestions
The key feature for a SPARQL language server is, in my opinion, code-completion.
Here the editor provides suggestions to the user.
In SPARQL this is not just a convenience. Without smart completion suggestsions a user has to know the ontology by heart. Its also a massive efficiency boost for experianced users.

Here is a possible structure for completions:

#### 1. Static

The simplest kind of completions is just to provide static snippets of keywords or structures to the user.
For this just the "Location" of the cursor is relevant, the context of the content of the knowledge-graph does not influence the suggestions.

Here are a few Examples i implemented:

|                                                |                                                |                                     |
| ---------------------------------------------- | ---------------------------------------------- | ----------------------------------- |
| ![](img/examples/cmp_select_pre.png)           | ![](img/examples/cmp_filter_pre.png)           | ![](img/examples/cmp_bind_pre.png)  |
| ![](img/examples/cmp_select_post.png)          | ![](img/examples/cmp_filter_post.png)          | ![](img/examples/cmp_bind_post.png) |

#### 2. Dynamic

Dynamic completions are more complex. They use all information availible to provide context based suggestions.

This again could be split into two categories:

#### 2.1. Offline

Here the Language server only uses "localy availible" information: 
That is: Text in the editor and the data bundled with the language server (for example known prefixes).

A Simple example for this are suggestions of already defined variables:

![](img/examples/cmp_variable.png)
This uses the parse-tree to find all variables.
Currently this is done very stupid, as this also suggests variables when they are out of scope:

![](img/examples/cmp_variable_dumb.png)

#### 2.1. Online

Here the Language Server uses data from a SPARQL-endpoint to provide completion suggestions.
>[!warning]
>This is not implemented yet!

Here is a example from the Qlever-OSM-endpoint: <htps://qlever.cs.uni-freiburg.de/api/osm-planet/>.
[**O**pen**S**treet**M**ap](https://www.openstreetmap.org/) (OSM) is a Project that collects Geodata that is publicly accesible. Basically Google maps, just without Google.
This data can be represented in a rdf-knowledge graph and queried using SPARQL.
For example this query returns every bicycle parking spot in [Freiburg](https://www.openstreetmap.org/relation/62768):
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

Now i want to know the same thing just for Berlin.  
Unfortunatly i forgot that the relation ID of berlin is `62422`...  
A good online-contextual-completion should help me:

|                            |                           |
| -------------------------- | ------------------------- |
| ![](img/examples/online-cmp-before.png) | ![](img/examples/online-cmp-after.png) |

# Using the Language server

Ok, now let's talk about how to use the language server.
A editor needs to have a language server client and connect to our server.

I looked at three editors: neovim, vs-code, and a custom web-based-editor.

## Neovim

To get a language server running with neovim is very easy because it has a build in language client and
neovim is build to be hacked.

### Installing the Programm

First `Qlue-ls` needs to be availible as executable binary on the system, neovim runs on.
You could just build the binary from source.
But to make it more convenient i added the binary into two "repositories":

The Rust repository [crate.io](https://crates.io/crates/qlue-ls). To install from there run:
```shell
cargo install qlue-ls
```
And the python repository [pypi](https://pypi.org/project/qlue-ls/). To install from there run:
```shell
pipx install qlue-ls
```
The python package is build with the help of [maturin](https://github.com/PyO3/maturin).

### Attaching

All you need is a `init.lua` file in you config directory with the following snippet:

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
When you open a `sparql` file (a file with suffix `.rq`).
This runs the comand `qlue-ls server` and connects the language client to this process.

## VS-code

In vs-code there is no build in language-client.
Instead you have to create a vs-code-extention that acts as a language-client.
I will do that in the Future(TM).

## The Browser

Okay, now the for the cherry on top:
Let's connect this thing to a web-based editor (a editor that runs in a browser).
"But wait!" you might say, "browser is javascript land".
And you would be right, until 2017.

### WebAssembly

[WebAssembly](https://webassembly.org/) is a open standard defined by the [World Wide Web Consortium](https://www.w3.org/).
It defines a Bytecode to run programms withing the browser.
All big browers support it.

If your Programm can be convertet (compiled) to this WebAssembly(WASM) bytecode it can executed in the Browser.
![](img/examples/WebAssembly-data-flow-architecture.png)[^9]
So now we need to build a compiler to convert Rust to WASM bytecode...
Unfortunatly [some strangers on the Internet](https://github.com/rustwasm/team) already did that.
The Project is called [wasm-pack](https://rustwasm.github.io/wasm-pack/).

It's very organic and simple, you just anotate the method or struct you want to "export" to wasm.

```rust
#[wasm_bindgen]
pub fn init_language_server(writer: web_sys::WritableStreamDefaultWriter) -> Server {
    wasm_logger::init(wasm_logger::Config::default());
    Server::new(move |message| send_message(&writer, message))
}
```

Then wasm-pack builds WASM-bytecode and javascript "glue code".
This glue code does a lot off stuff to bridge the gap between javascript and WASM.

```javascript
...
export function init_language_server(writer) {
    const ret = wasm.init_language_server(writer);
    return Server.__wrap(ret);
}
...
```

To actually run this in the browser i  needed to jump through a cuple more hoops but i spare you the details.
I packaged the result and uploaded it to [npm](https://www.npmjs.com/package/qlue-ls): a javascript repository.
Now we can install the package using npm and access it from a javascript file:

```bash
npm install qlue-ls
```

```javascript
import { init_language_server } from "qlue-ls";
const server = init_language_server(...);
server.listen(...);
```

#### Tree-Sitter in WebAssembly

As stated earlyer tree-sitter creates a parser in c and i wrote my programm in Rust.
Turns our compiling a Rust programm that calles external C functions to WebAssembly creates something
called "**ABI-Incompatabilities**".
I wish i could tell you how i solved that, but to be honest, i don't want to talk about this exerience, since it was extremly painfull.

In short i found a [project](https://github.com/shadaj/tree-sitter-c2rust) that ports the C code to Rust and all my issues diasappeard.
This is the second reason for me to move away from tree-sitter.

### The Editor

Next we need a web-based editor.
There are a cupple options, here are a few:

- [Monaco-Editor](https://microsoft.github.io/monaco-editor/)
- [CodeMirror](https://codemirror.net/)
- [ACE Editor](https://ace.c9.io/)

None of them have build in language clients, but all of them have extensions that provides one.

- Monaco-Editor: [monaco-languageclient](https://github.com/TypeFox/monaco-languageclient)
- CodeMirror: [@shopify/codemirror-language-client](https://github.com/shopify/theme-tools), [codemirror-languageserver](https://github.com/FurqanSoftware/codemirror-languageserver), [codemirror-languageservice](https://github.com/remcohaszing/codemirror-languageservice)
- ACE Editor: [ace-linters](https://github.com/mkslanc/ace-linters)

But to be fair the ones for CodeMirror don't really look stable.

I decided to go with Monaco for a cupple reasons:

- It's very mature (Its the same editor that drives vs-code)
- The language-client extension looks very active.
- I hope to get a synergy effect since both LSP and Monaco are from Microsoft

I will eventually also try ACE.

To get Monaco running with the language client i again had to jump to a few hoops (actually a lot),
but i again spare you with the details.

### TextMate

A good editor needs Syntax highlighting. (The thing that makes the colors)

| without syntag highlighting              | with syntax highlighting              |
| ---------------------------------------- | ------------------------------------- |
| ![](img/examples/highlighting_off.png)   | ![](img/examples/highlighting_on.png) |

For Monaco-Editor there are 2 options:

- [Monarch](https://microsoft.github.io/monaco-editor/monarch.html): build speciffically for Monaco, well suited for simple languages
- [TextMate Grammar](https://macromates.com/manual/en/language_grammars): feature of the [TextMate](https://macromates.com/) Editor, widely used standard in text editors 

I went with TextMate Grammar.
It's more complex but also more powerfull and can be used with other editors.

TextMate Grammars use the [Oniguruma](https://github.com/kkos/oniguruma) regex engine.
Basically you use write patters that identify the tokens you want to highlight.

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

Now this actually falls under "hoops i needed to jump through".
But i think its quite estetic.

Here are the Pieces we have so far:

The DOM-Element, the Monoco-Editor-Worker, the TextMate-Worker, the Language-Server.
With Worker i mean [Web Worker](https://de.wikipedia.org/wiki/Web_Worker). Web-Worker's allow javascript code to be executed separated from the main-thread.

All thats left is is a Worker that forwards the Language-server WASM component.

Here is the architekture:

![](img/setup.png)

# How does Qlue-ls compare against other software?

Now we have a language server we can use in various Editors that has limited capabilities.
How does this compare against other software thats out there?

Here is what i found:

| Tool                                                                                | Description                                                                                                               | Platform     | Mainainer                                                             | FOSS          |
| ----------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- | ------------ | --------------------------------------------------------------------- | ------------- |
| [sparql-langauge-server](https://github.com/stardog-union/stardog-language-servers) | SPARQL language server build in TypeScript                                                                                | web & native | [Stardog](https://www.stardog.com/)                                   | ✅  Apache-2.0 |
| [RDF and SPARQL plugin](https://plugins.jetbrains.com/plugin/13838-rdf-and-sparql)  | A RDF and SPARQL Plugin for JetBrains IDEs                                                                                | native       | [Sharedvocabs Ltd](https://plugins.jetbrains.com/vendor/sharedvocabs) | ❌ 5€/month    |
| [Qlever-UI](https://github.com/ad-freiburg/qlever-ui)                               | A custom [codemirror](https://codemirror.net/) editor for the [Qlever](https://github.com/ad-freiburg/QLever) triplestore | web          | [ad-freiburg](https://ad.informatik.uni-freiburg.de/)                 | ✅  Apache-2.0 |
| [YASGUI](https://github.com/TriplyDB/Yasgui)                                        | A lightweight, web-based SPARQL editor                                                                                    | web          | [Triply](https://triply.cc/en-US)                                     | ✅  MIT        |
| [sparql-formatter](https://github.com/sparqling/sparql-formatter/tree/main)         | A SPARQL formatter build in JavaScript                                                                                    | web          | [SPARQLing](https://github.com/sparqling)                             | ?             |

Let's look at how these tools compare in the aspects discussed in this article. (**personal opinion**)

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
| ⭐      | bearly working  |
| ❌      | not implemented |

>[!warning]
>These observations came from a brief inspection. It's possible they are better than i think they are!
>For example YASGUI also supports custom queries for completion. I did not have the time to test this propertly!
>I know that Qlever-UI also uses custom queries so the destiction between the two may be unfair.

## Qlue-ls vs sparql-formatter

Since [sparql-formatter](https://github.com/sparqling/sparql-formatter/tree/main) is "just" a formatter, its not really fair to compare it like the other tools.
So let's do a direct comparisson:

I build a scraper that collected all 381 example queries from the [Wikidata Query Service](https://query.wikidata.org/).
Then i formatted the query with Qlue-ls and sparql-formatter and comparred the two results using diff.
Here are the differences:

### Core differences

#### Error resilience
First of all sparql-formatter expects correct queries, while Qlue-ls also works with incomplete queries.
The quality of the result depends on how the parser handles the error.
#### Concrete vs Abstract
Then the two function slightly different.
Qlue-ls uses a Concrete Syntax Tree (CST) and sparql-formatter uses a Abstract Syntax Tree (AST).

While a CST represnt the **complete** syntactic structure of the input (ncluding parentheses, punctuation ...),
a AST is a abstracted representation. It omits syntactic details and only keeps the essentioal elements.

For example a input like `(22 * (3 - 1) - 2)`
The trees would be:

| CST          | AST          |
| ------------ | ------------ |
| ![](img/cst.png) | ![](img/ast.png) |

While you can use both for formatting, a CST guaranties that no token gets lost or is added.
For example sparql-formatter adds a "." here:

| before                     | after                     |
| -------------------------- | ------------------------- |
| ![](img/examples/format_ast_before.png) | ![](img/examples/format_ast_after.png) |

### Errors

Let's look at critical errors the formatters do (or dont do)

#### Errors in sparql-formatter

#### Strange Brackets in Expression lists

sparql-formatter throws a error for this query:
![](img/examples/format-sq-error1.png)
```
SyntaxError at line:5(col:11-12)
Expected: != && * + - / < <= = > >= IN NOT ||
--
          )

```
The brackets are strange but no reason to throw an error.

#### Encapsulated Agregate
sparql-formatter throws a error for encapsulated aggregates:

![](img/examples/format_error_3.png)

```
file:///usr/lib/node_modules/sparql-formatter/src/formatter.js:811
  if (variable.varType === '$') {
               ^

TypeError: Cannot read properties of undefined (reading 'varType')
```
#### Errors in Qlue-ls formatter

None i know of.

### Opinion based differences

#### Where-Clause newline

| Qlue-ls                   | sparql-formatter          |
| ------------------------- | ------------------------- |
| ![](img/examples/format_diff_1_ql.png) | ![](img/examples/format_diff_1_sf.png) |

#### Union-Clause newline

| Qlue-ls                   | sparql-formatter          |
| ------------------------- | ------------------------- |
| ![](img/examples/format_diff_2_ql.png) | ![](img/examples/format_diff_2_sf.png) |

#### Capitalize Keywords

sparql-formatter captializes some, but not all keywords.
Qlue-ls capitalizes all.

| Qlue-ls                   | sparql-formatter          |
| ------------------------- | ------------------------- |
| ![](img/examples/format_diff_3_ql.png) | ![](img/examples/format_diff_3_sf.png) |

#### Linelength based linebreaks

sparql-formatter will add a linebreak if the line get to long.

| Qlue-ls                    | sparql-formatter          |
| -------------------------- | ------------------------- |
| ![](img/examples/format_error_3_ql.png) | ![](img/examples/format_diff_4_sf.png) |
### Summary

sparql-formatter has linelengh based linebreaks, but has a few runtime errors, adds tokens (non breaking) and has inconsitent capitalization.

Qlue-ls is error resilent and has no known runtime errors.

Overall the two create surprisingly similar outputs.
I opened up an issue for the bugs i found so maybe in the future we will produce the same output (aside the cst/ast stuff).

I also wanted to do a performance comparisson.
But creating arbitrarily long random SPARQL queries is a sidequest i dont have the time for.

# Future work

Qlue-ls is right now (07-01-2025) still in the alpha phase.
It was my first Rust project and first contact with linked-data.
I'm proud of my work, but im sure there is a lot to improve!

Aside the code quality, efficency, and so on, here is the roadmap for this project.

## Stronger Parser

As stated earlyer Tree-Sitter was nice to get goning fast, but its not the right soloution for this problem.
I need a Parser that **deterministically** gives me the exact "location" within the parse tree for any position in the editor.
This could be achieved with a resilient "LL(1)" parser.
There is a [article](https://matklad.github.io/2023/05/21/resilient-ll-parsing-tutorial.html) by [Alex Kladov](https://github.com/matklad), the creator of [rust-analyzer](https://github.com/rust-lang/rust-analyzer) (**the** rust language server) that goes into detail on this topic. This will be the topic of my Bachelor Thesis.

## Query Sparql endpoint

Currently Qlue-ls it not firing any query's against a SPARQL-endpoint.
This will open a new world of cool Features and make this Language server acutally usefull.

There are three reasons i did not implement this yet:
1. I think this will open a box of cool new Problems i dont have the Time to solve (and i dont want to do it for free)
2. I currently ship to WASM **and** x86 targets, this makes this a bit more complex
3. This opens the box of **async** and i heard that is another level of complexity in Rust

## Enhance existing features

Formatting should try to maintain a maximal line length.
It could also try to keep things more consise.

Diagnostics and code-actions can be extended majorly.

I did not implement this since its mostly repetive extra work that does not bring any new insights.
Also, since all existing features depend on the parse-tree, i want to build a stronger parser first!
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
[^7]:https://www.geeksforgeeks.org/postorder-traversal-of-binary-tree/
[^8]: https://code.visualstudio.com/api/language-extensions/language-server-extension-guide
[^9]:https://www.researchgate.net/figure/WebAssembly-data-flow-architecture_fig1_373229823

