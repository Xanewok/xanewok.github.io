---
layout: post
title: Cargo compile routine
category: draft
---
* `compile_ws`
  * resolve final package dep graph (from source, with overrides, features, create a DAG)
  * `ops::compile_targets`
    * create `Unit`s out of targets
    * create `Context` && `JobQueue`
    * `compile`(enqueue `Unit`s of work)
      * check freshness by comparing `fingerprint`
      * prepare `Work` (closure) with underlying `rustc` call (which is unused for dirty package?)
        * prepare appropriate (deps) arguments
        * call (when `Work` is executed) `Executor` to compile
      * enqueue `Work` in `JobQueue` (using underlying `DependencyGraph`)
      * call `compile` for dependencies
    * `JobQueue::execute` - actually coordinate and compile
      * create helper thread for `jobserver` (gives out N tokens for maximally N running parallel jobs/lanes)
      * do the work via `drain_the_queue` inside `crossbeam::scope`
        * dequeue if possible onto local queue (as dequeue may dequeue multiple targets)
        * `run` the `Job` if queue not empty and we have jobserver token
          * fresh package runs the job directly, dirty package `scope.spawn`s it
          * ^ this actually executes `Work` prepared previously
          * note progress
    * collect compilation results + map
