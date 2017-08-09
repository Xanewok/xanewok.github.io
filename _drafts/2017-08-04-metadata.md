---
layout: post
title: Taking a closer look at Cargo metadata
category: draft
---

Cargo does a great job managing dependencies, overriding them, specifying different targets for your package and other relevant information. How does it track and use all the information to coordinate the build procedure?

## Metadata
Mentioned information that Cargo has is called metadata. A Cargo subcommand that retrieves this information, `cargo metadata`, has been implemented somewhat around Rust 1.8. When executed in a Cargo project, it returns a JSON structure that describes all the relevant metadata for the current project. The format is packed, so to get a better look at it, it's a good idea to run `cargo metadata | python -m json.tool` (requires Python 2.6+, copied from a [StackOverflow answer](https://stackoverflow.com/a/1920585)).

What does it look like? Here's an output for a bare Cargo project created with `cargo new example`:
```javascript
"packages": [
  {
    "dependencies": [],
    "description": null,
    "features": {},
    "id": "example 0.1.0 (path+file:///home/xanewok/example)",
    "license": null,
    "license_file": null,
    "manifest_path": "/home/xanewok/example/Cargo.toml",
    "name": "example",
    "source": null,
    "targets": [
      {
        "crate_types": [
          "lib"
        ],
        "kind": [
          "lib"
        ],
        "name": "example",
        "src_path": "/home/xanewok/example/src/lib.rs"
      }
    ],
    "version": "0.1.0"
  }
],
"resolve": {
  "nodes": [
    {
      "dependencies": [],
      "id": "example 0.1.0 (path+file:///home/xanewok/example)"
    }
  ],
  "root": "example 0.1.0 (path+file:///home/xanewok/example)"
},
"target_directory": "/home/xanewok/example/target",
"version": 1,
"workspace_members": [
  "example 0.1.0 (path+file:///home/xanewok/example)"
]
```
*(^ Should I just copy the schema from the link?)*<br>
Output schema is defined over at [doc.crates.io](http://doc.crates.io/external-tools.html#information-about-project-structure).
Two most important structures there are `"packages"` and `"resolve"`.

Referring to the crates.io schema, `"packages"` is a list of packages for the workspace, including declared dependencies. The list is obtained by initially resolving declared dependencies, also providing more information about relevant packages, such as specific resolved version of the package or discovered targets (e.g. whether it's *bin*/*lib*/*proc-macro* or other, where are the corresponding fetched source files etc.) for it.

On the other hand, `"resolve"` is the resolved dependency directed acyclic graph. This information allows to just build the project bottom-up since graph structure guarantees that building the dependencies will build all of them and allow to build the root package (or a set of root packages, if we're talking about workspace consisting of multiple crates).

*Describe here how exactly Cargo does that in code?*<br>
*Should I mention more details on Cargo uses the computed information and fetched/resolved packages to actually coordinate the build process?*
