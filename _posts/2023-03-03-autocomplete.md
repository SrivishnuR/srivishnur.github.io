---
layout: post
title: Building a robust code completion engine using ANTLR
date: '2023-03-03 14:01:04 -0500'
categories: blog
---
*The original blog link no longer exists after Airkit was acquired*

<img src="/assets/autocomplete/start_gif.gif" alt="typecheck" width="80%"/>

Airscript, Airkit’s programming language, is what allows for the multitude of powerful features our low-code platform offers. Writing our own programming language allows for a very powerful custom feature set (e.g. variable level metadata boxing which allows us to track sensitive information across the whole system – blog on this coming soon). However, it also comes with tradeoffs – namely, you ditch the years of support other established languages have accumulated. One such example of this is something we often take for granted – intelligent code completion (also known as IntelliSense for those using Visual Studio Code).

In this blog, we’ll guide you through how we built Aircomplete – a robust code completion engine for Airscript. We wanted to make this blog accessible to those without much experience in the field of programming languages, and so we’ll first start off with an example driven explanation of how programming languages work. We’ll then use these concepts to explain how code completion works in depth, talk a little about implementation, and then end off with some further tweaks and optimizations we added to Aircomplete.

# How code completion works
While a meaningful autocomplete system for the English language can be very complex due to its massive vocabulary and complex rulesets, code completion is relatively simple as in contrast, programming languages have well defined grammars and limited scopes in vocabulary. At a high level, a programming language can be seen as a definition for a state machine. A code completion engine walks that state machine given an input string in your language and outputs suggestions based on the outgoing edges from the final state.

<figure>
    <img src="/assets/autocomplete/full_artn.png" alt="typecheck" width="80%"/>
    <figcaption>Figure 1: A subset of an example language’s state machine</figcaption>
</figure>

For example, in figure 1, from a final state of 11 the only code completion option would be `RBRACKET`, or “]”.

However, before we dive any deeper, we require a general understanding of how programming languages work. We’ll do this in the next section by building a programming language for id validation. For those with programming language experience, we recommend skipping to the **Back to code completion** section.

# Building our own programming language
Let’s say we want to build a toy language to validate a list of ids based on some proprietary id validation logic. Example sentences in our language would look like `[]`, `[id1,];`, `[id1, id2,];`, etc. Ultimately, we want a function idArrayValidator that takes in a string in our toy language and returns true or false based on whether all the ids in the list are valid.

This task requires three major steps.

1. *Create a lexer* – We need to create a lexer based on a defined lexer grammar. A lexer converts our code string into an array of base units called tokens.
2. *Create a parser* – Now we create a parser based on a defined parser grammar. A parser validates an array of tokens and converts it into a useful form (usually a parse tree) for further processing.
3. *Create a visitor* – To derive something meaningful from our parse tree (i.e. for compilation, interpretation, or code completion), we need to then traverse our tree and implement said functionality. This traversal is usually done with a visitor, but other design patterns, such as a listener, also exist.

A grammar can be seen as a schema. In the case of a lexer grammar – a schema that defines what constitutes a token, and in the case of a parser grammar – a schema that defines what organizations of said tokens are valid.

While this flow might seem complex, this process is greatly simplified with modern tools called parser generators which can generate a lexer and parser based on a supplied grammar. These parser generators can also often generate the boilerplate for a visitor as well, but full implementation of the visitor is up to the user, as that is defined by the use case.

One of the most popular parser generators is [ANTLR](https://www.antlr.org/), and that’s what we’ll be using for the remainder of this blog. Now let’s start building our language!

## Create a lexer
Here is the ANTLR lexer grammar for our toy language (toyLexer.g4):

```
lexer grammar toyLexer;

channels {
 C_WHITESPACE
}

fragment F_LBRACKET: '[';
fragment F_RBRACKET: ']';
fragment F_COMMA: ',';
fragment F_SEMICOLON: ';';
fragment F_DIGIT: '0' .. '9';
fragment F_UPPER: 'A' .. 'Z';
fragment F_LOWER: 'a' .. 'z';
fragment F_LETTER: F_UPPER | F_LOWER;
fragment F_ID_CHAR: F_LETTER | F_DIGIT;
fragment F_ID: F_ID_CHAR+;

LBRACKET: F_LBRACKET;
RBRACKET: F_RBRACKET;
COMMA: F_COMMA;
SEMICOLON: F_SEMICOLON;
ID: F_ID;
```

Lines 7-16 define fragments – these are constructs to aid in readability and maintainability. Line 18-22 are the lines of importance – these define the tokens in our language based on the previously defined fragments.

When fed this grammar, ANTLR spits out a file named toyLexer.js, which we would then use to tokenize code written in our test language. Here are some examples:

* “[];” tokenizes to `[LBRACKET, RBRACKET, SEMICOLON]`
* “,][” tokenizes to `[COMMA, RBRACKET, LBRACKET]`
* “[ id1 ];” tokenizes to `[LBRACKET, ID, RBRACKET, SEMICOLON]`
* “[ id1, ];” tokenizes to `[LBRACKET, ID, COMMA, RBRACKET, SEMICOLON]`
* “!:” throws an error as `!` or `:` are not valid tokens defined in our grammar

## Create a parser
Here is the ANTLR parser grammar for our toy language (toyParser.g4):

```
parser grammar toyParser;

options {
 tokenVocab = toyLexer;
}

entry: LBRACKET identifier* RBRACKET SEMICOLON;
identifier: ID COMMA;
```

Lines 7 and 8 define the rules for our grammar. Lowercase words stand for other rules in the grammar while uppercase words stand for tokens. The asterisk (*) means 0 or more. The rule `entry` will be used as our entrypoint to the grammar.

So our grammar here defines a language in which valid sentences start with an open bracket (`LBRACKET`), have 0 or more identifiers (`ID`) followed by a comma (`COMMA`), and end with a closed bracket (`RBRACKET`) and semicolon (`SEMICOLON`). For more information on how parser grammars work we highly recommend reading up on extended Backus–Naur form (EBNFS). This [blog](https://tomassetti.me/ebnf/) is a useful resource.

When fed this grammar, ANTLR spits out a file named toyParser.js, which we could then use to validate our array of tokens and subsequently generate a parse tree. Here are some validation examples:

* `[LBRACKET, RBRACKET, SEMICOLON]` is valid
* `[COMMA, RBRACKET, LBRACKET]` is not valid
* `[LBRACKET, ID, RBRACKET, SEMICOLON]` is not valid as it is missing a `COMMA` token before `RBRACKET`
* `[LBRACKET, ID, COMMA, RBRACKET, SEMICOLON]` is valid

Our parse tree takes the form of a javascript object, but ANTLR provides us a very handy GUI tool called TestRig to aid in visualization. Here’s the parse tree for the string “[ foo, bar, ];”:

<figure>
    <img src="/assets/autocomplete/toy_parse_tree.png" alt="typecheck" width="80%"/>
    <figcaption>Figure 2: The parse tree for the string “[foo, bar,];”</figcaption>
</figure>

As we can see from the image, the leaves are always tokens, and the internal nodes are always rules.

## Create a visitor

Now that we have our parse tree, we need to create a visitor to validate our ids. Here is the file ANTLR generated for us (toyParserVisitor.js):

```
// Generated from toyParser.g4 by ANTLR 4.8
// jshint ignore: start
var antlr4 = require('antlr4/index');
// This class defines a complete generic visitor for a parse tree produced by toyParser.
function toyParserVisitor() {
 antlr4.tree.ParseTreeVisitor.call(this);
 return this;
}

toyParserVisitor.prototype = Object.create(antlr4.tree.ParseTreeVisitor.prototype);

toyParserVisitor.prototype.constructor = toyParserVisitor;

// Visit a parse tree produced by toyParser#entry.
toyParserVisitor.prototype.visitEntry = function(ctx) {
 return this.visitChildren(ctx);
};

// Visit a parse tree produced by toyParser#identifier.
toyParserVisitor.prototype.visitIdentifier = function(ctx) {
 return this.visitChildren(ctx);
};

exports.toyParserVisitor = toyParserVisitor;
```

A function is generated for each rule in the grammar, and the argument ctx points to the node in the parse tree the function is being invoked for. Right now all this visitor does is traverse the tree, but this boilerplate allows us to implement custom functionality based on which rule we’re located in. Let’s say we import a boolean function validateId(id: string). we can now implement our validation functionality by making the following modifications to visitIdentifier:

```
// Visit a parse tree produced by toyParser#identifier.
toyParserVisitor.prototype.visitIdentifier = function(ctx) {
 const id = ctx.ID() // Returns the string in the ID token in the rule identifier
 if(!validateId(id)) {
   throw new Error(`${id} is not a valid id!`);
 }

 return this.visitChildren(ctx);
};
```

Voila! Our toy language is complete! Let’s wire it all up into the idArrayValidator function:

```
const antlr4 = require("antlr4")
const toyLexer = require("./toyLexer").toyLexer
const toyParser = require("./toyParser").toyParser
const toyVisitor = require("./toyParserVisitor").toyParserVisitor

function idArrayValidator(input) {
   // Tokenization
   const chars = new antlr4.InputStream(input)
   const lexer = new toyLexer(chars)
   const tokens = new antlr4.CommonTokenStream(lexer)

   // Getting the parse tree
   const parser = new toyParser(tokens)
   const parseTree = parser.entry()

   // Visiting the parse tree
   const visitor = new toyVisitor()
   try {
       visitor.visit(parseTree)
   } catch {
       return false
   }

   return true
}
```

One issue with our code is that it returns false on both invalid input to the language as well as on invalid ids, but the fix is left as an exercise for our reader ;).

# Back to code completion
So what was the importance of all of this? Well, in the process of building a lexer and parser for our language, ANTLR also internally built an ARTN (augmented recursive transition network – essentially a DFA with support for state and recursion). In simpler terms, ANTLR built a state machine for our language. Using the array of tokens we received from the lexer, we can walk through this state machine, branching out as needed, and the outgoing edges of our set of valid end nodes will consist of our code completion options (valid end nodes consist of end nodes that are reached while consuming the entirety of the token array). Let’s highlight this with an example – for our toy language, the ARTN looks like:



