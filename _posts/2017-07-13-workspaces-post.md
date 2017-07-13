---
layout: post
title: 'Draft: Supporting workspaces in RLS'
category: draft
---
One of the main goals for my GSoC project was to increase the usability of RLS and the range of projects it supports. As I mentioned in the [previous]({% post_url 2017-07-06-working-on-rust-language-server-for-gsoc-2017 %}) post, I managed to implement a support for a common project layout, which is a single package split into library and a binary that uses it, however there still remains a larger goal in sight: multiple packages and with it, Cargo workspaces.

To briefly explain what it is - a workspace is a set of packages that all share the same Cargo.lock and output directory, according to the [second edition](https://doc.rust-lang.org/book/second-edition/ch14-03-cargo-workspaces.html) of the Rust Book. It can also define its own package and since workspaces cannot be nested, each member package can only belong to a single workspace.

## Designing the solution
When designing the new implementation, creating a design document helped me immensely. Not only it helped me lay down and materialize the design that existed only as a thought or an idea but it also serves as a reference as to why certain design decisions were made in the end, along with the context.
I will now try to describe the design in general and explain the rationale behind it.

As of now, RLS works in a single package mode. It will still resolve dependencies, generate all the intermediate build artifacts along with analysis data, for a given package, however after doing so it'll continue to only run the linked compiler for an active target in the package. What this means is that, depending on which target is configured as active in the RLS, it will collect diagnostic messages and check compilation only for that specific target. That's what currently `build_lib`, `build_bin` and partially `cfg_test` configuration options are for.

The end goal is to change how the RLS operates and to allow it to support multiple targets by default. This can be often desired even for a single package, like previously described project layout with both library and dependent binary target.

However, since it will fundamentally change how the RLS works, a complete switch to a final design in one move would not only be risky in terms of a possible regression or the implementation being buggy but it would also require substantial amount of work, where it would have to be constantly adapted to regular changes that are being done in codebase and it would not provide any features during that time until completion.

## The plan
That is why the work on supporting multiple active packages will be done in an incremental manner and initially gated behind a `workspace_mode` configuration switch, and the plan to do so can be briefly laid out out as follows:
1. First, a prototype implementation will be created that uses Cargo exclusively for every build and collects generated output from a compiler that is executed by Cargo
2. Next step will be to hijack the Cargo build procedure and in place of a Cargo-executed compiler RLS will run the linked one with a replaced file loader and analysis data passed-in memory along with diagnostics for improved performance
3. Finally, RLS should extract the dependency graph from Cargo and based on that, will fully coordinate the build on its own using the linked compiler like above

This could be even further extended to work with custom build systems, probably by consuming some sort of a build plan in a standard format (support for emitting one from Cargo is already [planned](https://github.com/rust-lang/cargo/issues/3815)) but that is probably outside the scope of the current project given time constraints. 

## And now the details
Additional `analyze_package` option will be provided to serve as a convenience option when working on and interested in only single workspace package (this could potentially decrease compilation times, depending on the workspace) and it will act as if `cargo check -p <analyze_package>` was executed.

#### Build Queue
There is one tricky bit once RLS starts running the linked compiler in-process instead of executing separate `rustc` processes by Cargo - *environment variables*. We want to parallelize the build as much as possible to improve the compilation times, however since it will be only run inside a single RLS process, we only have a single environment for it. An environment is essentially a global mutable state and changing it for every starting parallel build is just asking for trouble, since we cannot guarantee that it stays the same throughout the build (Cargo provides [certain guarantees](https://github.com/rust-lang/cargo/blob/master/src/doc/environment-variables.md) during compilation).

To deal with this, the initial, not-yet-cached build for the project will be run, as it would be in the prototype, in a parallel fashion because it's safe to assume that the cost of sequential compilation of the dependencies will outweight the benefit of receiving the results in-memory. Since most often the changes in a workspace and the structure of it won't necessarily require a rebuild of all packages, subsequent requested builds will be later run sequentially to ensure consistent environment during package compilation. However, as always it's best to profile first to measure the performance for both scenarios and only then decide on the final behaviour.

Once RLS has a dependency graph there is an obvious optimization that can be performed when modifying few dependent packages at once (e.g. when doing a bulk rename). Right now, after requesting a build RLS queues it and executes it after some delay (configurable via `wait_to_build` option), however when modifying multiple packages we only need to queue the bottom one, which pulls others that might have changed. There's no need to recompile a dependency, only for it to be recompiled again when a dependent package pulls it instantly after.

#### Files
Since we do not generate the dependency graph ourselves, we need to rebuild it via Cargo when a source file is added or deleted, or when a Cargo.toml file is modified. Moving files around, as well as modifying Cargo.toml, can change the workspace structure or even implicit targets (e.g. `src/main.rs` or `src/lib.rs`) and that's why the graph has to be rebuilt to avoid stale data. Once it's regenerated, RLS can continue as usual.

Cargo relies on the notion of registries and hash fingerprints to determine whether a package needs rebuilding. However, when files are modified in-editor, there are no changes on disk and so Cargo can't know that a certain package needs rebuilding. While RLS does provide a virtual file loader facility for the linked compiler, package freshness is checked by Cargo before the compiler has a chance to run. Providing and injecting our own virtual registry only to fake fingerprints, as well as modifying Cargo API only for RLS' purposes, seems a bit to extreme, however once RLS has a dependency graph itself the problem effectively solves itself. With that, it won't rely on Cargo to do the compilation and coordination anymore and it can already run the compiler itself with the virtual file system to provide the modified source files.

One more thing to add is that LSP servers aren't explicitly forbidden to perform disk I/O or watch the files themselves, however the protocol provides a `workspace/didChangeWatchedFiles` notification for whenever a file watched by the client changes and seems that watching on the client side is preferred, as per [this discussion](https://github.com/Microsoft/language-server-protocol/issues/237). With this, RLS won't (nor probably it should) do the file watching itself and will rely on the protocol messages instead.

#### Analysis
Publishing diagnostics and updating current analysis data can be done either per-crate or per-build. The main difference is the latency to user and possible order of build stages (compiling and processing analysis data).

Preferably we want to stream the data in a per-crate fashion. Thanks to this the user wouldn't have to wait for the complete compilation to finish to see the resulting diagnostics. In case the build will be run parallel, during the routine there should be a separate thread that processes incoming messages and analysis after each crate compilation and updates it on the fly until the build is finished.

# Work to come
Yesterday a [PR](https://github.com/rust-lang-nursery/rls/pull/409) was merged with the prototype implementation of the workspace support. So far I mostly tested it on the [webrender](https://github.com/servo/webrender) project and was thrilled to see it working! It's rough around the edges and opt-in via `workspace_mode` as I mentioned, but I'd like to encourage switching it on for people working on workspaces. Compilation times may not be perfect, however further testing and feedback will be greatly appreciated!

Now what's left are the next mentioned iterations, which will be mostly performance optimizations and moving to managing the package dependencies on the RLS side. I'm very excited to finally see some results of my work and I hope it will also prove useful to others. I'll be back with more updates on how the work is progressing. Stay tuned!