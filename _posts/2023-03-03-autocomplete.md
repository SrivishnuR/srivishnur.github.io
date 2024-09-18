---
layout: post
title: Building a robust code completion engine using ANTLR
date: '2023-03-03 14:01:04 -0500'
categories: blog
---
*The original blog link no longer exists after Airkit was acquired*

<figure>
    <img src="/assets/autocomplete/start_gif.gif" />
</figure>

Airscript, Airkit’s programming language, is what allows for the multitude of powerful features our low-code platform offers. Writing our own programming language allows for a very powerful custom feature set (e.g. variable level metadata boxing which allows us to track sensitive information across the whole system – blog on this coming soon). However, it also comes with tradeoffs – namely, you ditch the years of support other established languages have accumulated. One such example of this is something we often take for granted – intelligent code completion (also known as IntelliSense for those using Visual Studio Code).

In this blog, we’ll guide you through how we built Aircomplete – a robust code completion engine for Airscript. We wanted to make this blog accessible to those without much experience in the field of programming languages, and so we’ll first start off with an example driven explanation of how programming languages work. We’ll then use these concepts to explain how code completion works in depth, talk a little about implementation, and then end off with some further tweaks and optimizations we added to Aircomplete.

# How code completion works
While a meaningful autocomplete system for the English language can be very complex due to its massive vocabulary and complex rulesets, code completion is relatively simple as in contrast, programming languages have well defined grammars and limited scopes in vocabulary. At a high level, a programming language can be seen as a definition for a state machine. A code completion engine walks that state machine given an input string in your language and outputs suggestions based on the outgoing edges from the final state.

<figure>
    <img src="/assets/autocomplete/full_artn.png" />
    <figcaption>Figure 1: A subset of an example language’s state machine</figcaption>
</figure>

For example, in figure 1, from a final state of 11 the only code completion option would be `RBRACKET`, or “]”.

However, before we dive any deeper, we require a general understanding of how programming languages work. We’ll do this in the next section by building a programming language for id validation. For those with programming language experience, we recommend skipping to the **Back to code completion** section.

# Building our own programming language
Let’s say we want to build a toy language to validate a list of ids based on some proprietary id validation logic. Example sentences in our language would look like `[]`, `[id1,];`, `[id1, id2,];`, etc. Ultimately, we want a function `idArrayValidator` that takes in a string in our toy language and returns true or false based on whether all the ids in the list are valid.

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
    <img src="/assets/autocomplete/toy_parse_tree.jpeg" />
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

A function is generated for each rule in the grammar, and the argument `ctx` points to the node in the parse tree the function is being invoked for. Right now all this visitor does is traverse the tree, but this boilerplate allows us to implement custom functionality based on which rule we’re located in. Let’s say we import a boolean function `validateId(id: string)`. we can now implement our validation functionality by making the following modifications to `visitIdentifier`:

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

Voila! Our toy language is complete! Let’s wire it all up into the `idArrayValidator` function:

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

<figure>
    <img src="/assets/autocomplete/full_artn.png" />
    <figcaption>Figure 3: The ARTN for the rule entry</figcaption>
</figure>

<figure>
    <img src="/assets/autocomplete/rule_artn.png" />
    <figcaption>Figure 4: The ARTN for the rule identifier</figcaption>
</figure>

The nodes are states, while the edges are labeled with a token (called atomic transitions) or ε (called epsilon transitions). Epsilon transitions are added to aid in the programmatic generation of state machines and are meant to be skipped (any state machine with epsilon transitions can be turned into a functionally equivalent state machine without them).

Let’s say we wanted code completion for the string “[id1”. Sending this to the lexer would spit out the token array `[LBRACKET, ID]`. Now we walk the ARTN with this array of tokens.

1. We start at state 0 and skip to state 4 due to the epsilon transition.
2. We consume the first token (`LBRACKET`) to progress to state 8. Our token array is now `[ID]`.
3. At state 8, we have two branches – the left branch that enters the `identifier` rule and epsilon skips to state 14, and the right branch that epsilon skips to state 11.
	* At state 11, our token array is not empty, and the outgoing edge, `RBRACKET`, does not match the token in the token array, `ID`. As a result, this is not a valid end node, and this branch is terminated.
	* At state 14, we consume the last token in the array, `ID`, and move to state 15. Since we have no tokens left in our array, we add state 15 to our set of end nodes.
    
Now, we look at our set of end nodes and collect all the tokens associated with the outgoing atomic transitions, skipping through the epsilon transitions. Since our set of end nodes only contains state 15, we get the list `[COMMA]`.  We can then map this array of tokens to user friendly strings, and voila, we (almost) have code completion!

Why almost? Well, what user friendly string do we map to if one of our code completion options is `ID` (e.g. if our input is “[“)? In our toy language, we might have a list of valid ids we could suggest here, but with a non-trivial language, here is where we have to deal with scoping. Let’s use an example in javascript to highlight this.

```
let globalVar = true
[1]
{
 [2]
 let scopeVar = true
 [3]
}
[4]
```

Let’s say at each cursor position, 1-4, our code completion engine suggests an `ID` symbol. For the respective position, the valid `ID` mappings would be:

1. `globalVar`
2. `globalVar`, since we are before the definition of scopeVar
3. `globalVar` and `scopeVar`, since we are past the definition of `scopeVar`
4. `globalVar`, since `scopeVar` is out of scope

As a result, we can’t trivially return all defined variables when we see an `ID` symbol. We need to return suggestions based on the variable scope at the cursor position. Typically, the way a scoping system for a language is implemented is by utilizing the generated visitor, and if you’re building your own language, you most likely have a system built out which you can extend to generate the scope for code completion. There is a catch though – the code completion engine in the majority of cases deals with incomplete code, and as a result the scope generator also needs to be modified to be able to work with incomplete code.

# Implementation
So to build code completion, we first need to build an engine that can walk the ANTLR ARTN and retrieve us the code completion tokens, and then we need a mechanism to retrieve the scope at the cursor position. Finally, we need some code to wire this all together and output the options to the UI.

To walk the ANTLR ARTN, we heavily recommend using or adapting the [antlr4-c3 library](https://github.com/mike-lischke/antlr4-c3), specifically CodeCompletionCore.ts – we modified this library to work with the ANTLR4 javascript runtime here at Airkit. We won’t go into the details of implementing this library in this post, as this [blog post](https://tomassetti.me/code-completion-with-antlr4-c3/) does a great job of explaining the nuances.

To get the scope at the cursor position, one option would be to adapt SymbolTable.ts in antlr4-c3 as highlighted in the blog post above. At Airkit, we chose to instead build out a system that handles scope as a linked list, and we then extended the ANTLR generated visitor to generate the scope at our cursor position. The implementation of this system is a topic for a future blog.

Finally, to wire this all together, we ran our code completion and scope generator engines on our current input and then scraped the generated scope to get variable suggestions when `ID` is a valid code completion token. At the time of writing, Airkit’s expression editor is based on [monaco](https://microsoft.github.io/monaco-editor/), and so to surface suggestions to the UI we then registered a [completion item provider](https://microsoft.github.io/monaco-editor/typedoc/functions/languages.registerCompletionItemProvider.html), or a function that serves our valid code completion options based on the current context.

# Further optimizations and additions

## LL1 your grammar for better error correction
When basing your symbol table generation off of the parse tree generated by ANTLR, implicitly you’re relying on ANTLR’s error correction to parse out a meaningful structure when the input sentence is incomplete. However, without a bit of tweaking, this error correction may not always be as powerful as you want it to be. This is an issue we ran into at Airkit when dealing with code completion on array bindings, specifically index and union bindings. Index bindings are for your standard array data accesses (e.g. `arr[1]` would return the element at index 1), while union bindings are for when you want the data at a selection of array indexes (e.g. `arr[0, 2]` returns an array containing the elements at index 0 and 2).

We’ve built a simple grammar to highlight our issue.

```
entry: expression EOF;

expression: identifier childBinding?;

childBinding: bindableChild childBinding?;
bindableChild: LBRACKET (indexedBinding | unionBinding) RBRACKET;

indexedBinding: expression;
unionBinding: expression (COMMA expression)+;

identifier: ID;
```

Let’s say we have a global scope with the variable `foo` and the array `arr`. So for code completion on a string such as “arr[f”, we would expect one of our options to be the variable foo. However, the observed behavior was that our scope would not be generated properly and as a result we wouldn’t return any options for code completion. This was due to ANTLR’s error correction mechanism – it was outputting the following parse tree:


<figure>
    <img src="/assets/autocomplete/old_parse_tree.png" />
    <figcaption>Figure 5: The parse tree for the string “arr[f”</figcaption>
</figure>

The red box around “f” means its an error node. As a result, our `visitIdentifier` visitor function wasn’t run as expected and the scope wasn’t being generated.

Upon further inspection, we were able to attribute this to the layout of our grammar. What was happening was that during the generation of the parse tree, ANTLR would enter the `bindableChild` rule. It would then attempt to filter the potential options, `indexedBinding` and `unionBinding`, by peeking into the first element of the rule. However, both `indexedBinding` and `unionBinding` start with the exact same rule, `expression`, and as a result, ANTLR wasn’t able to differentiate between the two based on the information given. ANTLR then fails out and returns an error node for the “f”.

We were able to solve this with a simple refactor to the grammar.

```
entry: expression EOF;

expression: identifier childBinding?;

childBinding: bindableChild childBinding?;
bindableChild: LBRACKET expressionBinding RBRACKET;

expressionBinding: expression unionBindingExtension?;
unionBindingExtension: (COMMA union += expression)+;

identifier: ID;
```

The corresponding parse tree for “arr[f” based on this grammar is:

<figure>
    <img src="/assets/autocomplete/new_parse_tree.png" />
    <figcaption>Figure 6: The new parse tree for the string “arr[f”</figcaption>
</figure>

The difference is night and day – here we are able to generate a parse tree that enters `expression` and `identifier` and thus allows for a smarter scope generation which results in expected code completion options. This was all caused by refactoring the `indexedBinding` and `unionBinding` rules into one rule – `expressionBinding`. This `expressionBinding` rule combines all array binding rules that start with expression, and as thus, ANTLR is able to deterministically look ahead into `expressionBinding` and generate a more complete parse tree.

In more technical terms, what we did here was make our grammar LL1 (or left lookahead 1). This means our parser only has to look ahead one token to create a valid parse tree. However, we don’t recommend arbitrarily making your language LL1, this can potentially explode the size of your grammar and make it very confusing to work with. Instead, we recommend identifying where poor error correction affects functionality negatively and spot correcting. While our example grammar has been completely converted to LL1, this is not the case with the Airscript production grammar.

## Adding code completion to sample data
One major annoyance when dealing with 3rd party APIs in a high code language is that the response is often not typed – getting the specific field you want is a very manual process that involves constantly switching tabs between your editor and API documentation.

This is a situation in which Airkit shines. Due to the nature of Airkit Data Flows, each operation (such as a data transform or an http request) is separated to an individual frame which allows for a visually clean code flow design as well as operation isolated testing. While usually the output of each operation has a set type the code completion engine can utilize for suggestions in future frames, http requests are a prime example of when this isn’t the case. But in these situations, what we’re able to do instead is generate a type for you based on an example test run for the operation, and as a result we can give you full code completion functionality for the response of **any** api call!

For example, let’s say we wanted a cat fact. We’ll start by requesting a list of cat facts.

<figure>
    <img src="/assets/autocomplete/request_details.png" />
    <figcaption>Figure 7: The catfact HTTP request connection frame</figcaption>
</figure>

We then click run (shown below) to test the endpoint and generate sample data.

<figure>
    <img src="/assets/autocomplete/sample_data.png" />
    <figcaption>Figure 8: Running the frame once to generate sample data</figcaption>
</figure>

Now we want to isolate a single fact in a following transform operation. Normally, this is where we’d have to refer to the api documentation to understand the structure of the response. However, with Airkit, this is unnecessary as we have full code completion capabilities on the result by using the sample data!

<figure>
    <img src="/assets/autocomplete/end_gif.gif" />
    <figcaption>Figure 9: Full code completion action for the untyped api response</figcaption>
</figure>

Again, all of this is without any official support from the api devs!

# In conclusion
In this blog, we dove into the details of how programming languages and code completion engines work. We then talked about implementation and explored some of the more complex topics surrounding code completion. Ultimately, we hope that we were able to teach you something new, and if we’re lucky, maybe even inspire you to build a robust code completion engine of your own.
