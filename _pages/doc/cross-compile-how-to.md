---
title: "How to Cross-Compile M68k Binaries"
layout: single
---
In this page we're demonstrating how to cross compile source code for M68k using the LLVM toolchain.
> Please refer to [How to Build M68k LLVM](/doc/build-from-source) if you haven't build a m68k LLVM toolchain.

## Preparations
Modern programs oftenly require libraries -- static or dynamic -- during build-time. For instance, most of the programs on Linux nowadays link C library (`libc`) dynamically so you need `libc.so` somewhere compiler can find. Therefore, we need to prepare these necessary libraries for cross-compilation first.

Here are the instructions to install them:
### Ubuntu 20.04 x86_64
There are plenty of cross-compiling resources for m68k on mainstream x86_64 Ubuntu. You simply need to install the following packages:
```bash
sudo apt install libc6-m68k-cross \
                 libc6-dev-m68k-cross \
                 libgcc-9-dev-m68k-cross \
                 binutils-m68k-linux-gnu
```
If you want to compile C++ source code, be sure to install the stdc++ libraries, too:
```bash
sudo apt install libstdc++-9-dev-m68k-cross \
                 libstdc++6-m68k-cross
```

Alternatively, you can install m68k GCC & G++ cross-compiler bundle packages, which bring in all the aforementioned libraries:
```bash
sudo apt install gcc-m68k-linux-gnu g++-m68k-linux-gnu
```

## Cross-compiling
The commands are quite simple:
```bash
# Object file
clang -target m68k-linux-gnu -c input.c
# Executable
clang -target m68k-linux-gnu input.c -o input.m68k_exe
```
Please don't omit either the `linux` or `gnu` parts in the target triple, they are _equally_ important with the `m68k` part. Because they inform the compiler driver to find the previously-installed libraries at the correct path, which we need to manually specify otherwise.