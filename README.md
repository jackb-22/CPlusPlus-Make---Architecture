# mymake: C++ Make-Compatible Build System (Architecture Overview)

*Source code is private per Columbia University policy. This document outlines the system design and core implementation details.*

## Overview

`mymake` is a C++17 build tool that implements core `make` behavior: parsing build rules, resolving dependencies, applying implicit C/C++ compilation rules, checking file timestamps, executing shell commands, and avoiding unnecessary rebuilds.

Building it required solving four core problems: parsing a Makefile-like language, representing build rules as an executable dependency graph, deciding when targets are stale, and preserving parsed state to avoid reparsing unchanged build files.

---

## Core Systems

### Makefile Parsing

`mymake` reads a `MyMakefile` from the current directory and parses it line by line.

The parser recognizes four major forms:

- command lines beginning with a tab
- variable assignments such as `CC = gcc`
- rule declarations such as `target: dep1 dep2`
- variable references such as `$(CXXFLAGS)`

Parsing is implemented with `std::regex`, using separate patterns for command lines, variable assignments, rule headers, and variable expansion. Before parsing, comments are stripped and whitespace is normalized through helper functions for trimming and token splitting.

The parser also tracks the current active targets so that command lines following a rule are attached to every target declared by that rule, matching standard Makefile behavior.

---

### Rule Graph Representation

Each build target is stored as a `Rule` object inside an `unordered_map<std::string, Rule>`.

A rule contains:

- whether the target is `.PHONY`
- whether it has an implicit dependency
- an ordered list of commands
- an ordered dependency list
- a backing `unordered_set` to prevent duplicate dependencies

Dependencies are stored with both a `deque` and an `unordered_set`. The `deque` preserves build order, while the set guarantees uniqueness. Explicit dependencies are appended normally, while implicit dependencies are pushed to the front so generated source dependencies are checked before user-declared secondary dependencies.

The `Maker` object owns the full rule table and stores a pointer to the first real target, which becomes the default build target.

---

### Variable Expansion

The parser supports Make-style variable substitution using `$(VAR)` syntax.

A variable table is initialized with default compiler variables:

- `CC = cc`
- `CXX = c++`

As the parser encounters assignment lines, it updates the variable table. During parsing and implicit rule generation, variable references are replaced by their current values. Undefined variables expand to an empty string, keeping the parser simple and permissive.

---

### `.PHONY` Handling

`.PHONY` is treated as a special rule. When the parser sees:

```make
.PHONY: clean all
