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
biggest, in terms of scope and work, planned feature was to implement supporting
multiple package targets and packages in general, including Cargo workspaces.
Another, lesser, feature to explore was to implement previewing macro expansion
in the editor, however, in the end, there was not enough time to pursue that.

At the time of writing I managed to land 27/31 PRs, push 44 *(?)* commits (totaling
*(green)* +? insertions and *(red)* -? deletions) in total.

### Most notable PRs
* Supporting multiple packages and targets
  * Support analyzing a specific binary in project
    <br>\#373 by Xanewok was merged on 25 Jun
  * Support projects with both bin and lib crate types
    <br>\#363 by Xanewok was merged on 26 Jun
  * Workspaces
    <br>\#409 by Xanewok was merged on 13 Ju
  * Use linked compiler during Cargo `exec()` callback
    <br>\#424 by Xanewok was merged 9 days ago
  * Infer appropriate crate target from workspace
    <br>\#438 by Xanewok was merged an hour ago
  * Create simple dep-graph from Cargo metadata
    <br>\#441 by Xanewok was merged a day ago
* Smaller features and bugfixes
  * Provide CompletionItem.kind for Racer completions
    <br>\#237 by Xanewok was merged on 2 Apr
  * Set missing env vars from Cargo in a compilation step
    <br>\#337 by Xanewok was merged on 5 Jun
  * Prevent aggressive project rebuilding by retaining RUSTFLAGS order
    <br>\#349 by Xanewok was merged on 18 Jun
  * Opt-out of build after `initialize`
    <br>\#420 by Xanewok was merged 19 days ago
  * Ignore invalid file URI scheme in parse\_file\_path et al.
    <br>\#430 by Xanewok was merged 14 days ago
  * Don't ignore incomplete server config, use default values if needed
    <br>\#432 by Xanewok was merged 12 days ago

* Refactoring / project organization
  * Stubbed out separate simple test projects for tests
    <br>\#231 by Xanewok was merged on 27 Mar
  * LsService::handle\_message match replaced with macro
    <br>\#256 by Xanewok was merged on 1 May
  * Use ServerMessages in tests
    <br>\#323 by Xanewok was merged on 24 May
  * Use jsonrpc-core data for improved and more consistent error handling flow
    <br>\#402 by Xanewok was merged on 13 Jul
  * Miscellaneous, e.g. updating READMEs or project CI

###### Paragraph about work done for workspaces

###### Paragraph about troubles with with impl and in general

###### Last paragraph about takeaways and what I learned during GSoC/what would I recommend

