---
layout: post
title: One environment to rule them all

---
Since environment variables are [used heavily](https://github.com/rust-lang/cargo/blob/master/src/doc/environment-variables.md) by the Cargo and Rust compiler itself, RLS has to be very cautious and provide the same guarantees regarding the environment as OS does, for the duration of the underlying tool's execution. This means that the enviroment for Cargo or the compiler must stay the same or effectively be isolated or Bad Things™ may happen.

I was recently trying to switch the way the compiler is invoked for the diagnostics from running `rustc` process to running linked compiler and directly retrieving diagnostics and analysis data, while [working on supporting Cargo workspaces in RLS]({% post_url 2017-07-24-supporting-workspaces-in-rls %}). After the environment locking with a `Mutex` was [implemented](https://github.com/rust-lang-nursery/rls/commit/79d659e5699fbf7db5b4819e9a442fb3f550472a#diff-9997203f2de5b62d7810f98eebd0cb72R414), I encountered a strange regression for the workspace support test - the environment was effectively locked (somewhat) recursively, leading to a deadlock.

The initial implementation of the scope locking the environment with a `Mutex` used only a single lock. Originally, during the Cargo build routine, `rustc` compiler instances were ran as separate processes, each one having its own environment appropriately set by Cargo. However, executing the linked compiler lead to an attempt to lock the environment for the duration of the compilation, while it was still locked previously at the outer scope for the entire duration of Cargo execution. Oops.

What's needed was to provide a consistent and scoped environment at an outer scope, the Cargo execution, but also at the inner one, the compiler scope. Locking two separate `Mutex`es would not work, as we also support straight `rustc` build routine, where we don't invoke Cargo. Running both Cargo and `rustc` build routines in parallel could lead to an inconsistent environment - it's easy to imagine a scenario where there's a data race resulting in an inconsistent environment for one of the two.

The solution for the problem seemed like an easy one - all builds should first acquire a single, shared lock and whilst holding it, possibly acquire an inner one for the duration not longer than the first one is held for. It'd be great to encode that information using RAII guards with specified lifetimes of the lock, all while guaranteeing the order of locking, however there was one problem with this approach.

To intercept planned `rustc` calls, Cargo exposes an `Executor` trait, which effectively provides a callback (`exec()`) with all the arguments and environment `rustc` would be called, so the compiler execution can be freely customized. However, since API consumes a trait object `Arc<Executor>` it's not possible to limit the lifetime of an inner lock (since it'd have to live shorter than the outer lock).

At the moment, current implementation unfortunately does not strictly enforce lock order. The first, outer lock function returns a `(MutexGuard<'a, ()>, InnerLock)`, *(link to merged documentation?)* where InnerLock is a unit struct that has access to static inner lock and should not live longer than the returned outer lock guard. While technically it can be copied outside the scope of the initial lock guard, it seems acceptable for the time being, as it's required to further the work needed to support the workspaces in the RLS.

