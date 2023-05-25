---
title: "How to Build M68k LLVM"
---
Building M68k LLVM is roughly the same as building a normal LLVM. At the minimum:
```bash
$ git clone https://github.com/llvm/llvm-project.git
$ cd llvm-project
$ mkdir .build && cd .build
$ cmake -G Ninja -DLLVM_EXPERIMENTAL_TARGETS_TO_BUILD="M68k" ../llvm
$ ninja all
```
The only difference is the `LLVM_EXPERIMENTAL_TARGETS_TO_BUILD` cmake argument, since M68k is not an official target yet.

If you want to build Clang, make sure to add the `-DLLVM_ENABLE_PROJECTS="clang"` cmake argument as well. For more details, please refer to the [official document](https://llvm.org/docs/CMake.html).

Personally, I recommend the following cmake configurations for a development setup on Linux:
```bash
cmake -G Ninja -DCMAKE_BUILD_TYPE=Debug \
               -DBUILD_SHARED_LIBS=ON \
               -DLLVM_TARGETS_TO_BUILD=native \
               -DLLVM_EXPERIMENTAL_TARGETS_TO_BUILD="M68k" \
               -DLLVM_USE_LINKER=lld \
               -DLLVM_ENABLE_PROJECTS="clang;compiler-rt" \
               -DLLVM_USE_SPLIT_DWARF=ON \
               ../llvm
```
Of course, you need to install LLD for this configuration(or you can replace `lld` in the above command with `gold`), which is available in most of the Debian / Ubuntu distributions.
