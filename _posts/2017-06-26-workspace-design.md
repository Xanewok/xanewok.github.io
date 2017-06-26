---
layout: post
draft: true
title: Supporting workspaces in RLS (high-level design) (draft)
---

## Assumptions
### RLS on its own has to be as dumb as possible
We ideally shouldn't duplicate any code related to coordinating the build on our own.
<br>
Since Cargo is de facto *the* build system for Rust, we should just use that, rather than duplicate any functionality. By adding custom/duplicated logic to RLS we introduce a maintainability burden:
* Cargo behaviour can change over time,
* we have to be careful of possible discrepancies between standalone Cargo and RLS-invoked Cargo. 

### Build system logic should be transparent (without sacrificing configurability) for the user 
The less intermediate steps we introduce, e.g. custom/duplicated logic or entirely custom configuration, the better. Ideally there should be no difference for the RLS user in case he's using RLS or using standard, manual workflow. In general this would mean mimicking Cargo behaviour as closely as possible:
* inferring the same defaults/respecting workspace configuration (unless explicitly overriden)
* building everything in the same order/with the same flags

### Get it working for as many people as possible, first
Since [IDE support](https://github.com/rust-lang/rust-roadmap/issues/2) and [productivity in general](https://blog.rust-lang.org/2017/02/06/roadmap.html) has been the focus of 2017, it'd be good to provide a good baseline first and get RLS working for as most people as possible. Specifically I think we should aim for basic feature parity and consistency first, then focus on performance and possible QoL/nice-to-have features or niche use-cases.

## Proposal
The easiest way to achieve basic, broad and Cargo-like functionality is to run Cargo itself. That should provide a good baseline to compare with when working on further optimizations and improvements. This would be a good starting point for measuring benchmarks for RLS project analysis. Then, incrementally, we could support more features as we go.
#### Stage 1: by default run Cargo with RLS-linked compiler incl. diagnostics
Currently RLS runs Cargo once during startup to get dependencies, run necessary build scripts etc. Only for specified, primary crate RLS caches args/envs with which Cargo runs the compiler, then runs dummy `--emit=dep-info` compilation for that crate in its place followed by running the actual RLS-linked compiler for single crate type.
0. Provide a way for user to opt-in into the new, experimental behaviour
1. In this mode, run Cargo as a default way to get diagnostics for the project (this needs benchmarking, but initially Cargo itself should provide acceptable performance and manage dependency graph, along with fresh/dirty state of dependent crates)
2. Retrieve information which crates belong to the workspace via Cargo API (`TODO`)
3. For the crates being in the analyzed workspace, during `cargo` routine, run the linked compiler (currently only done [during](https://github.com/rust-lang-nursery/rls/blob/master/rls/src/build.rs#L496) `rustc`) to retrieve diagnostics.<br>
(`TODO:` Need to check if we can provide a reliable way to parse and pass Cargo-emitted rustc args to Compiler API)<br>
   With each compiler run on crate target type we'd merge emitted (like [here](https://github.com/rust-lang-nursery/rls/blob/master/rls/src/build.rs#L505)) error messages, after which we would [parse](https://github.com/rust-lang-nursery/rls/blob/master/rls/src/actions/mod.rs#L151) those, and publish final diagnostics (currently we already support [multi-file diagnostics](https://github.com/rust-lang-nursery/rls/blob/master/rls/src/actions/mod.rs#L37)) (`TODO:` investigate coalescing [Analysis](https://github.com/rust-lang-nursery/rls/blob/master/rls/src/build.rs#L90))

Having done that, we'd truthfully support most popular/basic `cargo check` routine.