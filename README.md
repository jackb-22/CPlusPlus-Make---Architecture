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
```

the listed dependencies are marked as phony targets in the rule table.

Phony targets bypass normal timestamp freshness checks. This allows targets such as `clean` and `all` to execute even when files with the same names exist in the working directory.

---

### Implicit Build Rules

If a target has no explicit command, `mymake` attempts to synthesize a build command from the file layout.

For object files, it checks for matching source files:

```text
foo.o -> foo.cpp
foo.o -> foo.c
```

For executable targets with no extension, it checks whether a corresponding object rule exists:

```text
foo -> foo.o
```

If no object rule exists, it can also compile directly from:

```text
foo -> foo.cpp
foo -> foo.c
```

Generated commands use the configured compiler variables and flags, including `CC`, `CXX`, `CFLAGS`, `CXXFLAGS`, `LDFLAGS`, and `LDLIBS`.

Implicit dependencies are inserted into the target’s dependency list, allowing the later rebuild logic to treat generated source dependencies the same way as explicit dependencies.

---

### Cache & Serialization

To avoid reparsing unchanged build files, `mymake` writes parsed rule data into `.mymake.cache`.

On startup, the cache is used only if:

- `.mymake.cache` exists
- `MyMakefile` exists
- the cache timestamp is newer than or equal to the `MyMakefile` timestamp
- the cached rule graph is not empty
- implicit dependency assumptions are still valid

The cache stores serialized target-rule pairs. A sentinel target named `__DEFAULT__` records the default build target. Custom stream operators serialize and deserialize:

- target names
- phony flags
- implicit dependency flags
- ordered dependency lists
- command lists

The cache is invalidated if the source layout changes in a way that would affect implicit rule generation, such as a source file appearing or disappearing after the cache was written.

---

### Dependency Execution

Build execution is recursive. Calling `make(target)` walks the dependency graph before executing the target’s own commands.

For each dependency, `mymake` attempts to build it first. The recursive call tracks a `seen` set of active targets. If a target is encountered twice in the current dependency path, the tool detects a circular dependency and drops that edge rather than recursing forever.

This gives the build engine three responsibilities at once:

- recursively build dependencies
- detect and report circular dependency paths
- propagate whether dependency work occurred

---

### Timestamp-Based Rebuilds

For non-phony targets, `mymake` uses `std::filesystem::last_write_time` to decide whether a target is already up to date.

A target is skipped when:

- the target file exists
- all dependency files exist
- every dependency is older than or equal to the target

If those conditions hold, `mymake` raises a nonfatal `TargetUpToDate` condition. If the target has no commands and no dependency work was needed, it raises `TargetNothingToDo`.

This separates normal build outcomes from fatal failures. “Up to date” and “nothing to do” are not treated as crashes.

---

### Command Execution & Exit Propagation

Commands are executed through `std::system`.

Before execution, each command is printed, matching the visible behavior of traditional `make`. After execution, the process status is inspected with `WEXITSTATUS`. If a command fails, `mymake` throws a `TargetCommandFailed` exception containing both the target name and the command’s exit status.

The top-level driver returns that same exit status, allowing `mymake` to integrate correctly with scripts, shells, and higher-level build workflows.

---

### Error Model

The project uses a custom exception hierarchy split into fatal and nonfatal build outcomes.

Fatal exceptions include:

- missing `MyMakefile`
- invalid syntax
- no real targets
- unknown targets
- failed shell commands

Nonfatal exceptions include:

- target already up to date
- nothing to do for a target

This design keeps the recursive build engine clean: expected build states can unwind through the same mechanism as real errors, while the top-level driver decides whether to print a message, continue, or terminate.

---

### Rule Graph Printing

The `-p` flag prints the parsed rule graph.

The output includes:

- the default rule first
- phony markers
- dependency lists
- attached commands

This makes the parser and implicit-rule engine inspectable without executing the build, which is useful for debugging rule resolution, variable expansion, and cache behavior.

---

## Architectural Summary

`mymake` is organized around a single `Maker` object that converts a Makefile-like source file into an executable dependency graph. The parser handles syntax, variables, phony targets, and implicit rule synthesis. The executor walks that graph recursively, uses filesystem timestamps to decide freshness, detects cycles, and runs shell commands only when necessary. A persistent cache stores the parsed graph to reduce startup work while still invalidating stale implicit dependency assumptions.

The result is a compact build system that recreates the core architecture of `make`: declarative rules, dependency-driven execution, timestamp-based rebuilds, and command orchestration.
