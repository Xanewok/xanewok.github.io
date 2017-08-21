---
layout: post
title: Wrapping up work on the RLS for GSoC
category: draft
---
It's time to wrap up the work done on the RLS for this year's edition of Google
Summer of Code. Time surely flew fast since the [first blog post](
{% post_url 2017-07-06-working-on-rust-language-server-for-gsoc-2017 %})!

In general, the goal of the project was to improve the user experience and
IDE support story for Rust. Apart from the regular bugfixes and QoL changes, the
primary goal and biggest in terms of scope feature was to implement supporting
multiple package targets and packages in general, including Cargo workspaces.
Another one, a stretch goal of sorts, was to implement previewing macro expansion
in the editor, however, in the end, there was not enough time to pursue that.

At the time of writing, 2 out of 3 stages of workspace support has been implemented,
I managed to land [27 PRs](#most-notable-prs), push
46 commits (<span style="color:green">+2,637</span> insertions and
<span style="color:red">-1,139</span> deletions) in total.

### Work done for multiple packages and package targets
For supporting multiple crate targets, one of the most useful implemented features
is ability to specify which target (`[lib]` or which `[[bin]]`) should be analyzed
by the RLS, as well as supporting analyzing bin targets that require the lib from
the same package. Additionally, the RLS itself can now detect which target should
be analyzed, if the user didn't explicitly specify it in the configuration.

Furthermore, a new `workspace_mode` has been added. When under it, the RLS doesn't
panic anymore when analyzing Cargo workspaces and happily provides all the diagnostic
data from the compiler for all the packages inside (assuming files are saved on
the disk), if a specific package hasn't been explicitly specified by the user
via the `analyze_package` option.

The work for workspace support was split into 3 stages:
1. First one was to introduce
a new, separate mode, under which the RLS would not crash and provide very basic
diagnostics.
2. Second one was to improve both the performance and support by executing
the linked compiler and gathering diagnostics and analysis data, which drives the IDE
features such as goto def, from every package in the workspace.
3. Third one, the biggest,
was to create and store a dependency graph of units of work (compiler calls), which
would allow to improve the latency of the analysis and provide support for in-memory
modified files.

Currently, the two first stages are completed, with the third stage still being actively
worked on.

### Difficulties
One of the things I struggled the most with was prioritisation of the work
I was tasked with. Even though the planned work was clearly defined, I tried to
find a balance between doing the work on a bigger, planned feature and implementing
smaller QoL changes or fixes that did ease/allow some people usage of the RLS.
In the end, I found myself frequently working on the side tasks longer than I
should've, effectively delaying the main job I was tasked with in the first place.

Having more freedom over one's work is both good and bad. There are no strict
*dreadlines* looming over you, however it's easy to lose yourself if you don't
organize your own work yourself. When working on some side projects I sometimes
used organization tools such as Trello or good ol' Post-Its and in the end, I
think I should've used it for this project. I managed to do most of the planned
work, but I think this would allow me to focus better on the task at hand and better
prioritise the work.

I very much feel like this is the area I need to improve on the most.

### Takeaways
Before I started working on an open source project, it seemed really daunting to
contribute to one. I'm not sure what's causing this, maybe it's stories about
traditional open source projects' core developers posting rants or aggressively
rejecting contributions.

However it turns out the open source community is very welcoming and helpful.
I think I've been bitten by the OSS bug by now and definitely find myself
contributing also to other projects thanks to working on one during the Google
Summer of Code.

I also have to give a shout out to the Rust community specifically here. When posting
issues, asking (sometimes silly) questions on the IRC or getting some required features
in Cargo for the RLS, people were always open and helpful every step of the way.

Speaking from a contributor's perspective, it means a lot when people take their
time to slow down and thoroughly explain as to why and how certain things work
the way they do, so I'm really grateful for that. It encourages further contribution
and makes you feel even better when doing that! After all, the work is not only useful
to yourself, but also to other people in general.

I also learned quite a lot on how IDEs may be structured and how they interact with
other tools/systems, most notably the compiler and a build system, to provide the desired
functionality. Previously I treated it as a black box, where it performed some
arcane compiler invocations and everything Just Worked&trade;. Now that I had the occasion
to work on one, it helped demistify the way a compiler and an IDE may work, both in general
and in tandem. It made me realize that a mere mortal such as myself can also work on it,
since most of the time people tend to treat it like some mystical, arcane arts that only
select people can work on.

### Closing words

It's easy to doubt yourself, whether one should even apply for GSoC because of
seeming lack of skill or whatever. However, one thing to keep in mind is that
the GSoC isn't some internship at a sweatshop, where you have to churn out code
as fast as possible to meet some impossible deadlines. Primarily, it's a valuable
opportunity with the focus on learning.
It shifts the way one thinks about the open source movement/projects and it helps
to grow as a programmer, both in terms of skill and communication required to do
the job.

I wholeheartedly recommend it and if there's a single thing I regret is that
I didn't do it sooner. If you find yourself thinking whether should you apply - **do it**!
You'll love every second of it.

### Most notable PRs
###### Supporting multiple packages and targets
* [(#373) Support analyzing a specific binary in project](https://github.com/rust-lang-nursery/rls/pulls/373)
* [(#363) Support projects with both bin and lib crate types](https://github.com/rust-lang-nursery/rls/pulls/363)
* [(#409) Workspaces](https://github.com/rust-lang-nursery/rls/pulls/409)
* [(#424) Use linked compiler during Cargo `exec()` callback](https://github.com/rust-lang-nursery/rls/pulls/424)
* [(#438) Infer appropriate crate target from workspace](https://github.com/rust-lang-nursery/rls/pulls/438)
* [(#441) Create simple dep-graph from Cargo metadata](https://github.com/rust-lang-nursery/rls/pulls/441)

###### Smaller features and bugfixes
* [(#237) Provide CompletionItem.kind for Racer completions](https://github.com/rust-lang-nursery/rls/pulls/237)
* [(#337) Set missing env vars from Cargo in a compilation step](https://github.com/rust-lang-nursery/rls/pulls/337)
* [(#349) Prevent aggressive project rebuilding by retaining RUSTFLAGS order](https://github.com/rust-lang-nursery/rls/pulls/349)
* [(#420) Opt-out of build after `initialize`](https://github.com/rust-lang-nursery/rls/pulls/420)
* [(#430) Ignore invalid file URI scheme in parse\_file\_path et al.](https://github.com/rust-lang-nursery/rls/pulls/430)
* [(#432) Don't ignore incomplete server config, use default values if needed](https://github.com/rust-lang-nursery/rls/pulls/432)

###### Refactoring / project organization
* [(#231) Stubbed out separate simple test projects for tests](https://github.com/rust-lang-nursery/rls/pulls/231)
* [(#256) LsService::handle\_message match replaced with macro](https://github.com/rust-lang-nursery/rls/pulls/256)
* [(#323) Use ServerMessages in tests](https://github.com/rust-lang-nursery/rls/pulls/323)
* [(#402) Use jsonrpc-core data for improved and more consistent error handling flow](https://github.com/rust-lang-nursery/rls/pulls/402)
* Miscellaneous, e.g. updating READMEs or project CI

