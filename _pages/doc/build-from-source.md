---
title: "How to Build M68k LLVM"
---

## For users of M68k LLVM

Building M68k LLVM is roughly the same as building a normal LLVM.

Perequisites:

- A Unix-like OS
- A git client
- A native C++ compiler
- CMake
- Ninja build system

CMake usage can be displayed with `cmake --help`.

We'll use a directory named `build` to store build files. This directory is already ignored by the `.gitignore` file, so it's assumed to be the standard build directory.

```bash
$ git clone https://github.com/llvm/llvm-project.git
$ cd llvm-project
$ mkdir build
$ cd build
$ cmake -G Ninja -D CMAKE_BUILD_TYPE="Release" -D LLVM_EXPERIMENTAL_TARGETS_TO_BUILD="M68k" -S ../llvm
$ cmake --build .
```

The LLVM CMake options which affect the build are [documented here](https://llvm.org/docs/CMake.html).

The main difference from a typical LLVM configuration is the `LLVM_EXPERIMENTAL_TARGETS_TO_BUILD` CMake argument, since M68k is not an official target yet.

`-G Ninja`

We'll use the Ninja build system, rather than the default (Make on Unix-like OSes.) Ninja is faster than Make.

`CMAKE_BUILD_TYPE` tells CMake if we want optimisations, debug symbols or LLVM assertions. `Release` enables optimisations, and disables both debug symbols and LLVM assertions.

`-S ../llvm`

Specifies `../llvm` as the source directory.

If you want to build Clang, make sure to add the `-DLLVM_ENABLE_PROJECTS="clang"` cmake argument as well.

For more details about the CMake build process and various options, please refer to the [official documentation](https://llvm.org/docs/CMake.html).

## For contributors to M68k LLVM

### Example 1: Configuring for general development

The following cmake configurations may be used for a development setup on Linux:

```bash
cmake \
    -D BUILD_SHARED_LIBS="On" \
    -D CMAKE_BUILD_TYPE="Debug" \
    -D LLVM_ENABLE_PROJECTS="clang;compiler-rt" \
    -D LLVM_EXPERIMENTAL_TARGETS_TO_BUILD="M68k" \
    -D LLVM_TARGETS_TO_BUILD="Native" \
    -D LLVM_USE_LINKER="lld" \
    -D LLVM_USE_SPLIT_DWARF="On" \
    -G Ninja \
    -S ../llvm
```

`-D BUILD_SHARED_LIBS="On"`

We'll build shared libraries rather than static libraries, to save some disk space.

You need to install LLD for this configuration (or you can replace `lld` in the above command with `gold`), which is available in most of the Debian / Ubuntu distributions.

`-D LLVM_USE_SPLIT_DWARF="On"`

This option can reduce memory usage at link-time.

### Example 2: Configuring for running of LLVM tests

```sh
cmake \
    -D BUILD_SHARED_LIBS="On" \
    -D CMAKE_BUILD_TYPE="RelWithDebInfo" \
    -D LLVM_ENABLE_ASSERTIONS="On" \
    -D LLVM_ENABLE_PROJECTS="" \
    -D LLVM_EXPERIMENTAL_TARGETS_TO_BUILD="M68k" \
    -D LLVM_TARGETS_TO_BUILD="" \
    -D LLVM_USE_SPLIT_DWARF="On" \
    -G Ninja \
    -S ../llvm
```

`-D CMAKE_BUILD_TYPE="RelWithDebInfo"`

We'll build the `RelWithDebInfo` configuration rather than `Debug`, as otherwise running tests can be quite slow.

We could alternatively build the `Release` or `MinSizeRel` configuration instead, but debug information can be useful.

`-D LLVM_ENABLE_ASSERTIONS="On"`

We'll enable LLVM assertions, in order to catch more errors during runtime. Specifying a configuration other than `Debug` normally disables these assertions.

`-D LLVM_ENABLE_PROJECTS=""`

We can build more quickly by removing unnecessary projects from the build. If you're working with Clang, specify `clang`.

`-D LLVM_TARGETS_TO_BUILD=""`

We'll remove all other targets from the build. This can speed up the build if we don't need the native LLVM target.
