---
layout: post
category: GSoC
title: Working on Rust Language Server for GSoC 2017
---
## Introduction
This year's edition of Google Summer of Code has begun on May 30th and I managed to enroll as a student! This means I'll spend the summer hacking away at Rust Language Server. This post will mark a series of posts about my experiences with GSoC and specifically working on the RLS, which I hope will prove interesting and provide some insight on how such work may look like.

## The project
My GSoC project is about working on and extending [**Rust Language Server**](https://github.com/rust-lang-nursery/rls/), which is a tool providing analysis for Rust projects. It is not an another IDE, but rather a standalone tool, which uses Language Server Protocol for communicating with different frontends (clients). This approach to creating analysis tools is fairly new, since the protocol used has been recently standarised and since then managed to gain a widespread adoption. This means that there's a variety of things to work on and that there's more freedom to experiment on the final design of the project.

## Language Server Protocol
The protocol has been created and standarized by Microsoft and, as explained in its [repository](https://github.com/Microsoft/language-server-protocol),
> The Language Server protocol is used between a tool (the client) and a language smartness provider (the server) to integrate features like auto complete, goto definition, find all references and alike into the tool.

It uses slightly extended JSON-RPC protocol to define request/response flow and specifies data structures and  supported set of requests that can be used during communication.

The core idea is to extract and standarize the glue used to integrate all the language features into any text editor (which supports the protocol, that is). One really cool part about it is that for each language there is only need for one server implementation, significantly reducing the amount required to support every editor.

By providing a common language for clients and servers, it allows for a clear separation of concerns. With this the text editors can focus on something they were primarly designed for - editing text. Same goes for analysis tools being able to focus on providing quality and well-defined set of tools for working with a language, without having to worry about specific editor integration.

More information, along with a more detailed list of available client and server implementations (including RLS!), can be found over at [langserver.org](http://langserver.org/).


## Why not just develop an IDE or a plugin?
Most importantly, because it's a costly and time-consuming endeavour. Writing a fully-fledged, mature IDE from scratch takes a lot of focused effort. Doing so also often entails writing an internal compiler to provide or facilitate detailed language analysis. However since Rust is a complex language, which encodes a lot of information at compile-time (e.g. complex generics, trait system), writing such a compiler would require a huge amount of work or would lead to inaccuracies in analysis.

What if we instead choose to write an analysis plugin for one of the editors we prefer? The amount of work required will be somewhat less than to develop a complete IDE, however we still have to tailor our solution to a specific editor, which might also mean being limited by it, depending on our choice. Furthermore, since such a tool would be, more often than not, bound to the editor, by doing so we leave out users of editors, e.g. Vim or Sublime.

I'd like to also note that end goal isn't to shrink IDEs in terms of capabilities. Rather, it can be thought of as splitting an IDE into components, where LSP is just a custom inter-process protocol, used to communicate between the components.

## Architecture
Underlying protocol is defined using JSON-RPC

## Sunshine and rainbows?
It certainly doesn't mean we should just abandon existing tools and switch to protocol-based ones as soon as possible. For instance, I personally can't imagine myself using something different than Visual Studio for serious C++ development. While LSP provides an easily extensible communication interface, it currently tries to support the least common denominator in terms of language support.
<br>However, despite being relatively young, it is very much in active development and well-maintained. Every protocol version improves on itself and provides more community-requested features. With all that in mind, I think that designing Rust Language Server around the LSP was a good decision. It currently gives the best bang for the buck and will only get better in the future.
