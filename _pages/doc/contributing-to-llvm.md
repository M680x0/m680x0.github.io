---
title: "Contributing to LLVM"
---

## Prerequisites

First, read the [official documentation](https://llvm.org/docs/Contributing.html).

## Setup

### Forking and cloning the LLVM repository

Head over to [the LLVM repository page](https://github.com/llvm/llvm-project) and fork it.

Once GitHub has finished forking the repo, use git to clone your fork.

```bash
$ git clone git@github.com:<your-github-username>/llvm-project.git
$ cd llvm-project
$ mkdir build
$ cd build
```

## Building

See [How to Build M68k LLVM](build-from-source).

## Running the M68k tests

### Running tests using CMake

#### Running all the tests

Invoke CMake to run all the tests:

```bssh
cmake --build . --target check-all
```

#### Running specific sets of tests

It takes significantly longer to run all the tests than it does to only run targeted M68k-related tests. In order to iterate more quickly, you can run only the M68k tests, rather than all the tests.

```bash
cmake --build . --target check-llvm-codegen-m68k
```

There are several useful test-related targets:

| Target                            | Directory containing tests  |
| --------------------------------- | --------------------------- |
| `check-llvm-codegen-m68k`         | `test/CodeGen/M68k`         |
| `check-llvm-debuginfo-m68k`       | `test/DebugInfo/M68k`       |
| `check-llvm-mc-disassembler-m68k` | `test/MC/Disassembler/M68k` |
| `check-llvm-mc-m68k`              | `test/MC/M68k`              |

Additionally, the `llvm-test-depends` target builds all the dependencies required to run LLVM tests, but does not run the tests.

### Running tests using `llvm-lit`

Invoke CMake to build the testing dependencies:

```bash
cmake --build . --target llvm-test-depends
```

Invoke `llvm-lit` to run the M68k tests:

```bash
bin/llvm-lit \
    test/CodeGen/M68k \
    test/DebugInfo/M68k \
    test/MC/Disassembler/M68k \
    test/MC/M68k
```

`llvm-lit` is the [LLVM Integrated Tester](https://llvm.org/docs/CommandGuide/lit.html) utility.

We specify the directories containing M68k-related tests. The directories are relative to the path specified by `config.llvm_src_root` in `build/test/lit.site.cfg.py`.

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

## Making changes

**TO DO**

## Getting your changes into LLVM

See the [LLVM code review process documentation](https://llvm.org/docs/CodeReview.html).
