---
layout: post
title: 'Draft: Supporting workspaces in RLS'
category: draft
---
One of the main goals for my GSoC project is to increase the usability of the RLS and the range of projects it supports. As I mentioned in the [previous]({% post_url 2017-07-06-working-on-rust-language-server-for-gsoc-2017 %}) post, I managed to implement support for a common project layout, which is a single package split into library and a binary that uses it, however there still remains a larger goal in sight: multiple packages and with it, Cargo workspaces.

To briefly explain what it is, using second edition of The Book as a [reference](https://doc.rust-lang.org/book/second-edition/ch14-03-cargo-workspaces.html), **workspace** is a set of packages that all share the same *Cargo.lock* and output directory. Workspaces helps organize multiple related packages in a single repository, mainly by keeping a single and well-defined dependency set for all of them (by having a single *Cargo.lock* file) and allowing to share and reuse resulting build artifacts for the packages (by having a single, shared output directory).

## Designing the solution
When designing the new implementation, creating a design document helped me immensely. Not only did it help me lay down and materialize the design that existed only as an idea but it also serves as a reference as to why certain design decisions were made in the end, along with the context.

As of now, RLS works in a single package mode. Using Cargo, it still resolves dependencies, generates all the intermediate build artifacts along with analysis data for a given package, however after doing so it continues to only run the linked `rustc` compiler for an active target in the package. What this means is that, depending on which target is configured as active in the RLS, it will collect diagnostic messages and check compilation only for that specific target. That's what currently `build_lib`, `build_bin` and partially `cfg_test` configuration options are for.

The end goal is to change how the RLS operates and to allow it to support multiple targets by default. This can be often desired even for a single package, like previously described project layout with both library and dependent binary target.

However, since it will fundamentally change how the RLS works, a complete switch to a final design in one move would not only be risky in terms of a possible regression or the implementation being buggy but it would also require substantial amount of work, where it would have to be constantly adapted to frequent code changes and would not provide any features until completion.

## The plan
That is why the work on supporting multiple active packages will be done in an incremental manner and initially gated behind a `workspace_mode` configuration switch. The plan to do so can be briefly laid out out as follows:
1. First, a prototype implementation is created that moves away from running mostly linked `rustc` to using Cargo exclusively for every build. After every package compilation, generated output and analysis data file is consumed to build the analysis database.
2. From there we can use Cargo only for build coordination and run linked `rustc` compiler instead of the Cargo one. Thanks to this, the RLS will be able to provide more accurate analysis on the fly by feeding in-memory file buffer contents to the compiler, as well as fetching analysis data directly from memory instead of serializing and reading it from file.
3. Finally, RLS will coordinate builds on its own, at first using the dependency graph from Cargo. By doing so, it stops being directly dependent on Cargo and can be further extended to work with other build systems. Additionally, build queue and management can be more fine-grained, leaving room for more optimizations and reducing analysis latency.

Supporting custom build systems later on could probably be done by consuming some sort of a build plan in a standard format (support for Cargo emitting one is already [planned](https://github.com/rust-lang/cargo/issues/3815)), but that is probably outside the scope of the current project given time constraints. 

## And now the details
After specifying a high-level plan, it's time to delve into more details. These are not strictly tied to a certain step in the plan and mostly explain some design motivations and various caveats involved.

The compilation times can be quite high when analyzing every package in the workspace, and so additional `analyze_package` option will be provided to serve as a convenience when working on and interested in only single workspace package and will act as if `cargo check -p <analyze_package>` was executed.

#### Build Queue
There is one tricky bit once RLS starts running the linked compiler in-process instead of executing separate `rustc` processes by Cargo - *environment variables*. We want to parallelize the build as much as possible to improve the compilation times, however since it will be only run inside a single RLS process, we only have a single environment for it. An environment is essentially a global mutable state and changing it for every starting parallel build is just asking for trouble, since we cannot guarantee that it stays the same throughout the build (Cargo provides [certain guarantees](https://github.com/rust-lang/cargo/blob/master/src/doc/environment-variables.md) during compilation).

To deal with this, the initial, not-yet-cached build for the project will be run as it would be in the prototype, in a parallel fashion. In general, very often the full project build will include many directly unrelated dependencies. Because of this, it makes sense to compile them all in parallel and then suffer the cost of serializing and reading analysis data file for every package than to compile the packages one by one but with directly read analysis data in-memory.

However, having built necessary dependencies, subsequent changes to the workspace won't necessarily require a rebuild of all packages. This means that it'll probably be acceptable to run requested buids in a sequential manner, since we need to ensure consistent environment during package compilation.

Nevertheless, it's best to profile first to measure the performance for both scenarios and only then decide on the final behaviour.

#### Files
Since we do not generate the dependency graph ourselves, we need to rebuild it via Cargo when a source file is added or deleted, or when a Cargo.toml file is modified. Moving files around, as well as modifying Cargo.toml, can change the workspace structure or even implicit targets (e.g. `src/main.rs` or `src/lib.rs`) and that's why the graph has to be rebuilt to avoid stale data. Once it's regenerated, RLS can continue as usual.

Cargo relies on the notion of registries and hash fingerprints to determine whether a package needs rebuilding. However, when files are modified in-editor, there are no changes on disk and so Cargo can't know that a certain package needs rebuilding. While RLS does provide a virtual file loader facility for the linked compiler, package freshness is checked by Cargo before the compiler has a chance to run. Providing and injecting our own virtual registry only to fake fingerprints, as well as modifying Cargo API only for RLS' purposes, seems a bit to extreme, however once RLS has a dependency graph itself the problem effectively solves itself. With that, it won't rely on Cargo to do the compilation and coordination anymore and it can already run the compiler itself with the virtual file system to provide the modified source files.

One more thing to add is that LSP servers aren't explicitly forbidden to perform disk I/O or watch the files themselves, however the protocol provides a `workspace/didChangeWatchedFiles` notification for whenever a file watched by the client changes and seems that watching on the client side is preferred, as per [this discussion](https://github.com/Microsoft/language-server-protocol/issues/237). With this, RLS won't (nor probably it should) do the file watching itself and will rely on the protocol messages instead.

#### Analysis
During the build routine, publishing compiler diagnostics and updating current analysis data can be done at two different points in time. It can be done directly after each crate compilation or in one fell swoop only after the build is finished. Processing data after each crate instead of doing it all at once reduces latency to user but requires more work to keep analysis data consistent when mixing old and new analysis.

Preferably we want to stream the data in a per-crate fashion. In case the build will be run in parallel, during the routine there should be a separate thread that processes incoming messages and analysis after each crate compilation and updates it on the fly until the build is finished.

# Work to come
Few days ago a [PR](https://github.com/rust-lang-nursery/rls/pull/409) was merged with the prototype implementation of the workspace support. So far I mostly tested it on the [webrender](https://github.com/servo/webrender) project and was thrilled to see it working!

It still has few drawbacks when compared to current single package mode, most notably longer compilation times and requires file changes saved to disk for a correct analysis. It's still rough around the edges and may not work perfectly but I'd like to encourage switching it on for people working with Cargo workspaces. It's currently opt-in via `workspace_mode`, any additional testing and feedback will be greatly appreciated!

With the prototype done, now what's left is to improve it and slowly move towards coordinating the build ourselves and being less reliant on Cargo. I'm very excited to finally see some results of my work and I hope it will also prove useful to others. I'll be back with more updates on how the work is progressing. Stay tuned!