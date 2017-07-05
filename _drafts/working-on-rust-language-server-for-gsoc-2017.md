---
layout: post
category: GSoC
title: Working on Rust Language Server for GSoC 2017
---
## Introduction
This year's edition of Google Summer of Code has begun on May 30th and I managed to enroll as a student! This means I'll spend the summer hacking away at Rust Language Server. This post will mark a series of posts about my experiences with GSoC and specifically working on the RLS, which I hope will prove interesting and provide some insight on how such work may look like.

## The project
My GSoC project is about working on and extending [**Rust Language Server**](https://github.com/rust-lang-nursery/rls/), which is a tool providing analysis for Rust projects. It is not an another IDE, but rather a standalone tool, which uses Language Server Protocol for communicating with different frontends (clients). This approach to creating analysis tools is fairly new, since the protocol used has been recently standarised and since then managed to gain a widespread adoption. This means that there's a variety of things to work on and there's more freedom to experiment on the final design of the project.

## Language Server Protocol
The protocol has been created and standarized by Microsoft and, as explained in its [repository](https://github.com/Microsoft/language-server-protocol),
> The Language Server protocol is used between a tool (the client) and a language smartness provider (the server) to integrate features like auto complete, goto definition, find all references and alike into the tool.

The core idea is to extract and standarize the glue used to integrate all the language features into any text editor (which supports the protocol, that is).
It uses slightly extended JSON-RPC protocol to define request/response flow and specifies supported set of requests that can be used during communication.

What's really cool is that for each language there is need for only one server implementation, significantly reducing the amount required to support every editor.
By separating the core language service from the editor, the analysis tool can focus on providing quality and well-defined set of tools for working with a language, and the text editors on something they were primarly designed for - editing text.



More information, along with a more detailed list of available client and server implementations (including RLS!), can be found over at [langserver.org](http://langserver.org/).


## Why not just develop an IDE or a plugin?
Most importantly, because it's a costly and time-consuming endeavour. Writing a fully-fledged, mature IDE from scratch takes a lot of focused effort. Doing so also often entails writing an internal compiler to provide or facilitate detailed language analysis. However since Rust is a complex language, which encodes a lot of information at compile-time (e.g. complex generics, trait system), writing such a compiler would require a huge amount of work or would lead to inaccuracies in analysis.

What if we instead choose to write an analysis plugin for one of the editors we prefer? The amount of work required will be somewhat less than to develop a complete IDE, however we still have to tailor our solution to a specific editor, which might also mean being limited by it, depending on our choice. Furthermore, since such a tool would be, more often than not, bound to the editor, by doing so we leave out users of editors, e.g. Vim or Sublime.

I'd like to also note that the end goal isn't to shrink IDEs in terms of capabilities. Rather, it can be thought of as splitting an IDE into components, where LSP is just a custom inter-process protocol, used to communicate between the components.

## Architecture
Thankfully, Rust Language Server does not try to reimplement all the functionality on its own. Most importantly, it uses core Rust tools: `rustc`, the official compiler, and `cargo`, the Rust package manager. Cargo is used to detect project workspace and its configuration, while the analysis data is emitted by the compiler during a special *check* compilation, where the final building stage is omitted.<br>
RLS mostly coordinates the build process with the help of Cargo, and manages analysis data for diagnostic purposes and request handling.<br>
Finally, LSP is used to communicate between the RLS and a language server client, and the client is responsible for forwarding requests made by the user, as well as getting the information back, in the editor.

## Planned work and beyond
As I mentioned earlier, there's a variety of stuff that can be done for the RLS. My main focus will be on extending the integration with Cargo, specifically supporting project workspaces, which are used for managing bigger projects consisting of multiple packages, in the RLS. Big projects are where IDEs shine brightest, so bringing support for that would mean a considerable leap in terms of usability for the RLS. Having done that, I'd like to improve the ergonomics and user-facing part in general. One exciting feature to add would be macro expansion previewable in the editor.

All in all, I'm very excited to be working on this project. It's a great opportunity to learn more about inner workings of Rust and IDEs in general, and to do something usable by others. I always felt that Rust lacks much needed IDE support, so now's my chance to change that.
More to come, stay tuned.
