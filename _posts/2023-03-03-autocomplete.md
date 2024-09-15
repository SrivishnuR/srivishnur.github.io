---
layout: post
title: "Building a robust code completion engine using ANTLR"
date: 2023-03-03 14:01:04 -0500
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

However, before we dive any deeper, we require a general understanding of how programming languages work. We’ll do this in the next section by building a programming language for id validation. For those with programming language experience, we recommend skipping to the *Back to code completion* section.
