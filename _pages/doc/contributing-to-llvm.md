---
title: "Contributing to LLVM"
---

## Context

Intended audience:

- Those who want to contribute improvements or fixes to the LLVM M68k backend
- Experienced computer users
- Experienced C++ programmers
- Those familiar with git, GitHub and the command line

## Setup

Perequisites:

- A Unix-like OS
- A git client
- A native C++ compiler
- CMake
- Ninja build system
- A C++ code editor

### Forking and cloning the LLVM repository

Head over to [the LLVM repository page](https://github.com/llvm/llvm-project) and fork it.

Once GitHub has finished forking the repo, use git to clone your fork.

```sh
git clone git@github.com:<your-github-username>/llvm-project.git
cd llvm-project
mkdir build
```

### Generating build files using CMake

Invoke CMake to generate the build files:

```sh
cmake \
    -B build \
    -D BUILD_SHARED_LIBS="On" \
    -D CMAKE_BUILD_TYPE="Release" \
    -D LLVM_ENABLE_ASSERTIONS="On" \
    -D LLVM_ENABLE_PROJECTS="" \
    -D LLVM_EXPERIMENTAL_TARGETS_TO_BUILD="M68k" \
    -D LLVM_TARGETS_TO_BUILD="" \
    -D LLVM_USE_SPLIT_DWARF="On" \
    -G Ninja \
    -S llvm
```

#### Rationale for the CMake options

CMake usage can be displayed with `cmake --help`.

The LLVM CMake options which affect the build are [documented here](https://llvm.org/docs/CMake.html).

`-B build`

We'll use a directory named `build` to store build files. This directory is already ignored by the `.gitignore` file, so it's assumed to be the standard build directory.

`-D BUILD_SHARED_LIBS="On"`

We'll build shared libraries rather than static libraries, to save some disk space.

`-D CMAKE_BUILD_TYPE="Release"`

We'll build the `Release` configuration, as otherwise running tests can be quite slow.

**TODO:** Should we build the `RelWithDebInfo` configuration instead?

`-D LLVM_ENABLE_ASSERTIONS="On"`

We'll enable assertions, in order to catch more errors during runtime.

`-D LLVM_ENABLE_PROJECTS=""`

We'll remove unnecessary projects from the build.

`-D LLVM_EXPERIMENTAL_TARGETS_TO_BUILD="M68k"`

The M68k target is "experimental", meaning it's not built by default. We need to explicitly enable it.

`-D LLVM_TARGETS_TO_BUILD=""`

We'll remove all other targets from the build.

`-D LLVM_USE_SPLIT_DWARF="On"`

Remove memory pressure during link-time.

`-G Ninja`

Use the Ninja build system, rather than the default (Make.) Ninja is faster than Make.

`-S llvm`

Specify `llvm` as the source directory.

## Building LLVM for testing

Invoke CMake to build:

```sh
cmake --build build --target test-depends
```

The `test-depends` target contains all the dependencies required to run tests.

## Running the M68k tests

It takes significantly longer to run all the tests than it does to just run the M68k-related tests. In order to iterate more quickly, you can run only the M68k tests, rather than all the tests.

Invoke `llvm-lit` to run the M68k tests:

```sh
./build/bin/llvm-lit \
    llvm/test/CodeGen/M68k \
    llvm/test/DebugInfo/M68k \
    llvm/test/MC/Disassembler/M68k \
    llvm/test/MC/M68k
```

`llvm-lit` is the [LLVM Integrated Tester](https://llvm.org/docs/CommandGuide/lit.html) utility.

We specify the directories containing M68k-related tests.

You should see output something like:

```
-- Testing: 168 tests, 8 workers --
PASS: LLVM :: CodeGen/M68k/Arith/lshr.ll (1 of 168)
PASS: LLVM :: CodeGen/M68k/Alloc/dyn_alloca_aligned.ll (2 of 168)
(more tests)
PASS: LLVM :: MC/M68k/pc-rel.s (168 of 168)

Testing Time: 7.61s

Total Discovered Tests: 168
  Passed: 168 (100.00%)
```

## Running all the tests

Invoke CMake to run all the tests:

```sh
cmake --build build --target check-all
```

## Making changes

**TODO**

## Opening a Pull Request

**TODO**

## Getting the Pull Request reviewed

**TODO**

## Getting the Pull Request merged

**TODO**
