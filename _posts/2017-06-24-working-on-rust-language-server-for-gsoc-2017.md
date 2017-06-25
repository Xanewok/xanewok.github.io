---
layout: post
title: Working on Rust Language Server for GSoC 2017
---
## Introduction
This year's edition of Google Summer of Code has begun on May 30th and I managed to enroll as a student! This means I'll the spend summer hacking away at Rust Language Server.
<br>Having finally taken care of my exams I now have managed to find some time to write a blog post about my work. This post will mark a series of posts about my experiences with GSoC, open source development and working on RLS, specifically. I hope it'll prove interesting and provide some insight on how such work may look like.

## The project
My GSoC project is about working on and extending [**Rust Language Server**](https://github.com/rust-lang-nursery/rls/), which is a tool providing analysis for Rust projects. The key part is that it's not an another IDE or a plugin designed specifically for one, but rather it's a standalone tool using Language Server Protocol for communicating with different frontends (clients).
<br>This approach is different from the usual one of creating a dedicated development environment for a specific platform/language, each including its own broad and unique set of tools that facilitate programmer's work.

## Why not just develop an IDE (plugin)?
So why not just go the established route - create a dedicated IDE or a plugin for one? Most importantly, because it's a costly and time-consuming endeavour. Writing a fully-fledged, mature IDE from scratch takes a lot 
of focused effort. What if we choose to write an analysis plugin for one of the extensible editors that we fancy, instead? That's going to work, however we'll have to tailor our solution to the tool we're extending, and depending on which one we chose, we might even be limited by it. Furthermore, by targeting a specific editor we're leaving out users of other editors, e.g. Vim or Sublime. So can we have a cake and eat it too? 

## Language Server Protocol
That's where the [**Language Server Protocol**](https://github.com/Microsoft/language-server-protocol) comes in.
Being initially created and open-sourced by Microsoft, as explained in its repository,
> The Language Server protocol is used between a tool (the client) and a language smartness provider (the server) to integrate features like auto complete, goto definition, find all references and alike into the tool.

It attempts to extract and standarize the glue used to integrate all the language features into any text editor (which supports the protocol, that is). This approach gained significant traction and numerous LSP client and server implementations can be found listed (including RLS) at [langserver.org](http://langserver.org/).
<br>By providing a common language for clients and servers, it allows for a clear separation of concerns. With this the text editors can focus on something they were primarly designed for - editing text. Same goes for analysis tools being able to focus on providing quality and well-defined set of tools for working with a language, without having to worry about specific editor integration.

## Sunshine and rainbows?
It certainly doesn't mean we should just abandon existing tools and switch to protocol-based ones as soon as possible. For instance, I personally can't imagine myself using something different than Visual Studio for serious C++ development. While LSP provides an easily extensible communication interface, it currently tries to support the least common denominator in terms of language support.
<br>However, despite being relatively young, it is very much in active development and well-maintained. Every protocol version improves on itself and provides more community-requested features. With all that in mind, I think that designing Rust Language Server around the LSP was a good decision. It currently gives the best bang for the buck and will only get better in the future.
