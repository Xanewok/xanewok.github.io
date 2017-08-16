---
layout: post
title: Taking a closer look at Cargo metadata
category: draft
---
In this blog post I'm going to describe the Cargo metadata format and how it's
used to build a project.

Cargo does a great job managing different kinds of dependencies. It can easily
resolve those, taking into account different package features and targets to
understand how a project needs to be build. I'll use the word package to describe
a Cargo crate and by project I mean either a single or multiple packages.

You can use `cargo metadata` command to retrieve the metadata for the project.
When executed in a
Cargo project, it prints a JSON structure that describes all the relevant data
Cargo could resolve for the project. The format is packed, so to get a better
look at it, it's a good idea to run `cargo metadata | python -m json.tool`
(requires Python 2.6+).

Today we're mostly concerned with `packages` and `resolve` objects.

## Packages
The `packages` object is a list of `Package`s that are in the project scope, i.e.
are a dependency (direct or indirect) to the packages in the workspace.

Each `Package` contains further information about its own direct, declared
`Package` dependencies along with the SemVer requirement and the kind (whether
it's a regular or a `build-`/`dev-`dependency). Furthermore, it also contains
specified features for the package or other additional metadata, such as the
package name, description or a license.

## Resolve
Since declared dependencies specify SemVer constraints, these need to be further
resolved, in order to create a simpler, acyclic dependency graph with concrete
and locked down versions of the packages, that will be used during the build
procedure.

A `resolve` object is such a graph. The data representation is a
dependency graph between `PackageId`s. A `PackageId` is essentially a triple of
a package (name, exact version, source). This is not directly included in the
graph representation, but the `resolve` object is constructed with specified
features being taken into account, and also more information about a certain
package can be retrieved via the `PackageId` from the `packages` object.

This information allows to build the project bottom-up, starting from the
dependencies, and guarantees that after doing so, all required dependencies will
be built for the packages in the project.

### Examples?


###### Footnotes
Exact `cargo metadata` JSON schema is defined over at [doc.crates.io](http://doc.crates.io/external-tools.html#information-about-project-structure).

