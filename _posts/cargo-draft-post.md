---
layout: post
published: false
title: Modular approach of RLS
---

### Previously
As I mentioned [earlier]({% post_url 2017-06-24-working-on-rust-language-server-for-gsoc-2017 %}) (TODO: update previous post)...

Traditionally IDEs were heavily integrating their own custom set of tools. Most of the time this meant having its own custom implementation of the compiler and build system.
### Reimplementation
Apart from the fact, that each implementation probably has to be integrated with a specific editor (explained previously):
* Duplicates code/effort
* Introduces inconsistencies between the actual compiler and the diagnostic IDE compiler
  * Best case you get build status discrepancies in IDE regarding (IDE reports no errors but actual compiler does and vice-versa)
  * Worst case IDE unintentionally misinforms the user, which may introduce silent bugs (e.g. wrong overload chosen for code compiled by the actual compiler)

### Consistency
When it comes to developing software, one of the most important things must be consistency.
If the user has certain expectations with regards to how the result will be compiled/built, the less surprising
the outcome is the better.
<!-- After all noone likes to spend their Friday afternoon playing Sherlock and trying to
investigate why **this** line goes through a certain code path, while *obviously* **that** one doesn't, but it should. -->
<br>
It's reassuring to know that the tool only tries to help and automate the tedious bit without introducing its own version of things.

RLS doesn't try to be as smart as possible on its own. Under the hood it uses the same tools one would normally use for regular development:
* rustc - compilation/diagnostics
* cargo - build system
* rustfmt - style formatting
* clippy (Soon&trade;) - linting