---
layout: post
title: Workspaces - design doc
---
# Workspaces
A *workspace* is a set of packages that all share the same *Cargo.lock* and output directory. It can also define its own package. Workspaces cannot be nested and each member package can have only one direct parent workspace, prioritizing closest one up.

# Design Proposal/Rationale
In addition to current, single crate mode, this introduces a separate **workspace mode**.<br>
New behaviour will at first be opt-in, to isolate experimental work and make sure this will not introduce any regression to the current behaviour.<br>
Initially the priority is to implement actual support for workspaces and lay foundation for easier future improvements. This will be done by off-loading all build work to Cargo, specifically workspace discovery, generating dependency graph and coordinating crate compilation with diagnostics. Then the prototype will be improved in an incremental manner, providing a good baseline for regression testing.

## Settings
In general, the new mode would introduce 2 new settings to the rls.toml file:
- `workspace_mode` (bool, default: false)
- `analyze_package` (String, default: "") - which package to analyze inside current workspace

As mentioned above, the new behaviour would be opt-in via the `workspace_mode` flag, while also supporting single-package projects. Those can be thought of as a workspace with no members and its package being the workspace root one. Specifying `analyze_package` allows to analyze only given package, as if `cargo` was ran with `check -p <analyze_package>` passed, when at workspace root level. If RLS is initialized in a workspace-less or a workspace member package, `analyze_package` setting will be ignored. Otherwise if provided package can't be found in the workspace, user will be warned and no analysis will be performed.

## Build Queue
At first Cargo will do the heavy lifting regarding managing crate dependencies, detecting fresh/dirty state of crates, so this part will not require much change. RLS will still wait an appropriate amount of time, i.e. `WAIT_TO_BUILD`, until actual building will take place, which will be then further done by only calling `cargo()` inside  actual `build()` routine ([here](https://github.com/rust-lang-nursery/rls/blob/7d8bd56c065064e39730aa2f540cb5d23f777ceb/src/build.rs#L256)) and waiting for the whole workspace/package build to finish.

Later, assuming we would know which file corresponds to which crate and which files are relevant to the build, as well as knowing the current dependency graph, build coordination could work as follows:
1. On file modification we check if it belongs to the workspace and if it does:
   - mark the file/crate as dirty, along with every other dependent on it
   - add top level dep graph subtree root to the build queue
2. If during `WAIT_TO_BUILD` delay user will modify another dependent crate, then:
   - reset the timer on related subtree
   - detect if it changes (in case some higher level crate has changed) and add the top level root to the queue again.
3. If a file/crate is modified and dependent tree is currently being rebuilt, then it needs to wait until it is finished building.
4. Unrelated subtrees can be put separately in the queue.

## Build routine
### How?
For `cargo()` build routine, inside `exec` [callback](https://github.com/rust-lang-nursery/rls/blob/7d8bd56c065064e39730aa2f540cb5d23f777ceb/src/build.rs#L297), we'll first check if the crate is in the workspace and is specified to be analyzed. Since Cargo will want to execute `rustc` program with its own arguments and environment, we need to convert arguments using compiler API and ensure that environment matches. We should run compiler with in-memory passed analysis and safely set environment per each crate compilation or spawn different rustc processes and pass the diagnostics/analysis via pipe/disk.

Later when it'll be possible to compile certain crate types using cached dependency graph, each crate in the graph would have to cache its args and envs with which Cargo tried to compile it. After each respective crate compilation using only compiler, analysis data will be reloaded in a similar fashion to how it'll be done by the `cargo()` routine.
### Why?
**To consider:** Build threads will be run in parallel and surely environment will wildly differ between crates, as different crate types have their own environment set during compilation (as specified [here](https://github.com/rust-lang/cargo/blob/master/src/doc/environment-variables.md)). This means that eventually running rustc, ourselves or not, will probably have to be in a separate process with dedicated environment. Unfortunately this probably means that we won't be able to pass data in-memory (which implies a significant (de)serialization and I/O overhead) and more sophisticated way of communication will have to be chosen.
* **TODO:** Settle on how compiler will be run to satisfy different environments in parallel context.

## Analysis
### How?
Publishing diagnostics and updating current analysis data can be done either per-crate or per-build. The main difference is latency to user and possible order of build stages (actual building and post-processing analysis data).

First we would process diagnostics and analysis per-build. It simplifies the process, as we'd only need to clone old state for the scope of new build and coalesce diagnostics messages and either update local analysis after each crate compilation or store it and do it as a separate phase after the build. Then, after the build is done running, we'd swap current and resulting analysis data, as well as publish whole diagnostics. If an internal failure was detected (as represented by `BuildResult::Err`), then the user would be notified but no other measures would be taken.

Later on we could 'stream' the data in per-crate fashion. This would decrease latency to user by effectively streaming in diagnostics and analysis data. This means that ideally during build routine there should be a separate thread that processes incoming messages and analysis after each crate compilation and updates it on the fly until the build is finished.

Additionally, in case Cargo encounters an internal failure, it should still continue building as long as it can, what it can (something along the lines of `continue parse-after-error`, omitting every crate depending on the one causing a failure) and still feed relevant diagnostics and analysis to the RLS. User should still be notified of the failure.
### Why?
For per-build analysis processing, when directly reloading the data inside `exec` callback we could potentially stall and lengthen the build process, as processing analysis data takes some time, during which another dependent crate build could be executed. Hence why there should be a separate thread from the compiling ones, that would update analysis and publish diagnostics.

In case of encountering an internal failure, discarding the data for affected crates seems more natural than to use stale data, which may introduce possible inconsistencies or confuse the user.
## File system
### What/How?
Initially at least a simple file watching will have to be implemented in the RLS to catch any relevant changes files affecting the build. At first this could be all .rs and .toml files in the workspace of which current package is a member of. This feature should be gated behind an appropriate setting (like `enable_file_watching` in rls.toml), to allow for only manually requested builds and to reduce implicit building, which may be undesirable due to more intensive overall CPU usage.

More sophisticated solution, assuming we have information about relevant files to the compilation, would be to watch all those relevant files (e.g. files used by build script or dependencies outside current workspace) and auto-start compilation on any change.

### Why?
In general, majority of tools, which LSP was designed in mind with, prioritize currently opened buffers (represented by `didOpen`->`didChange`->`didClose` cycle [in the LSP](https://github.com/Microsoft/language-server-protocol/issues/237#issuecomment-299408816)) over what's in the filesystem but otherwise stay true to it. There is, however, one tricky bit about how LSP works with regards to workspaces and files in general. It is currently unspecified which files should client watch and if it can at all (further discussion at [#237](https://github.com/Microsoft/language-server-protocol/issues/237)). Additionally the client sents an `initialize` request with `rootUri` (or now deprecated `rootPath`) indicating a root of a workspace. There are few problems with this:
* Technically a protocol-conformant client can not inform RLS at all about .toml, currently-not-opened .rs or other file changes, because it's incapable or doesn't know what's relevant, leading to stale state
* Relying solely on client notifications means shifting the burden of understanding workspaces and relevant files to multiple, different client implementations
* External changes made outside the LSP workspace affecting it (e.g. in scope of Rust workspace, when client is initialized in a member directory) will probably not be detected, even if the protocol were to be further specified to allow for watching files specifically inside the workspace

This means that a reasonable and fairly future-proof solution would be to implement file watching within RLS itself.<br>
Another very important reason to have that is when files are added/changed/removed. Those changes are vital in how the project is eventually laid out and compiled, so it's essential to implement that, more so when an LSP client is incapable of file watching in general.

The only bit to watch out for are future changes to the semantics of workspace in which LSP server is initialized. If the protocol was to constrain file operation/notification to workspace under `rootUri`, then the possible tracking of relevant files outside the specified workspace will not be permitted. In this specific case initializing client to a member directory inside would also not be permitted and RLS should present a warning to the user and disallow it.

## Cargo support/consistency
For working with workspace and discovery thereof, we'll stay consistent with how Cargo is configured and for its configuration we'll mimick `cargo`'s [default initialization](https://github.com/rust-lang/cargo/blob/master/src/cargo/util/config.rs#L81) and set `cwd` to `build_dir` instead of `rls'`s `env::current_dir()` [here](https://github.com/rust-lang-nursery/rls/blob/master/src/build.rs#L613). This is required for package detection, since `Config::cwd()` is later propagated and used in package/workspace root discovery by Cargo.

### Vfs (unsolved)
#### What?
Currently Cargo doesn't expose a way to provide a virtual/in-memory `Source` or `Filesystem`, which proves to be a problem, since it won't know about file changes in the currently-opened file buffers. To solve that, one way would be to implement something like `InMemoryRegistry` implementing `RegistryData`, for new `Kind::InMemoryRegistry` to be used with along with `Source(Id)` (check [`LocalRegistry`](https://github.com/rust-lang/cargo/blob/master/src/cargo/sources/registry/local.rs#L20)and [`load()`](https://github.com/rust-lang/cargo/blob/master/src/cargo/sources/registry/local.rs#L38) for reference). However Cargo would have to know to use it, so one could probably try and replace every in-memory package manifest in the workspace (as initialized by RLS) with some replace mapping, but... that smells like a giant hack. :(

* **TODO:** Come up with a sensible approach on how to feed files in memory for Cargo, both in the internals and the API

#### Why?
Even if it was easily possible to map currently opened files to the ones on disk (i.e. when using compiler API in `exec`, which also probably won't be possible because of the parallel threads and different envs for compilation jobs, mentioned earlier), there still remains another problem. There may be pending changes in currently opened and unsaved files in the editor that would change how the workspace/package build is resolved, and so we must feed the in-memory representation of the files earlier than when everything is resolved and we only intercept `rustc` jobs in `Executor`.

# Proposed course of action
#### Prototype
To achieve a sufficiently working prototype, for every previous design proposal section, initial proposed change will be implemented at first.<br>
At the very least appropriate settings in rls.toml will be implemented and behaviour split between current mode and workspace mode. In the latter, RLS will try to run `cargo` with linked compiler, collecting diagnostics and analysis data for relevant crates in the workspace and finally updating it and publishing diagnostics as the last step. This also assumes correct detecting of the workspace and setting `build_dir` appropriately, along with `cwd` for Cargo config.

Next step would be to adapt Vfs to work with Cargo, which means modifying Cargo itself and figuring a way to elegantly weave in the virtual in-memory registry API. This seems like a good thing to do second, since keeping modified files open in the editor is a common scenario and Cargo/build routine should be aware of that to improve consistency and quality of life.

After that's done, or while waiting on the Cargo changes due to reviews, merges and similar, basic file watching spanning entire workspace (.rs/.toml files) would be a good step forward from there. Scenarios where files are changed out-of-editor are also very common (e.g. using a source control tool, adding/renaming/deleting files, even in editor).

#### Further improvements
##### Streaming diagnostics per-crate
With prototype finished, a low hanging fruit might be to process diagnostics/analysis data per crate, rather than per whole build. This would require separating a processing thread for the duration of `cargo` from the compilation jobs, not to stall the build itself. It would also decrease the time user would need to wait for diagnostics.
##### Fetching and maintaining dependency graph
Having a dependency graph on the RLS side would allow us to run lightweight Cargo-like builds ourselves. This will require changes to Cargo (return `DependencyGraph` or similar) and build queue and routine (explained above). Arguments and environment would be cached during `cargo()` build routine and build management would rely on dependencies 
##### Detecting relevant files for build routine
When user changes only contents of existing .rs files inside the workspace, without altering its structure (e.g. adding/renaming/deleting files) and configuration (changing .toml files), it's probably safe to assume that Cargo build routine will remain the same, so there's only need to check which files are modified to see which and in what order crates should be rebuilt with previous arg/env configuration from last Cargo run.
**To consider:** what about adding `extern crate <another>` in currently opened file buffer? Does it change how compilation is invoked for related crates?
Otherwise if structure or configuration of a workspace has changed, Cargo will have to be run again to detect current configuration and crate dependencies.

This could somehow be further expanded upon with detecting and retrieving a list of all the relevant files during the Cargo build. Requesting a build on changes to those files would allow analysis to more faithfully match the state of the workspace, without having to wait for an eventual Cargo rebuild.

#### Other ideas
* Would it be feasible/possible to extend compiler API and, in addition to passing args, pass overriden envs to be used for in-process compilation? (need to profile this, but this'd reduce overhead of IPC communication with rustc processes and I/O in general, related to save analysis data)
* Maybe a lower-level Cargo API, in addition to `compile_*` functions, like `compile_with_dep_graph` could prove useful. In addition to `Workspace` and `Config`, user could also pass `DependencyGraph` or similar with specified packages' freshness and source, allowing for bypassing overhead of target/dependency collection, checking fingerprints etc. and specifying a custom, in-memory source for those packages (useful for temporary buffered files in the editor)