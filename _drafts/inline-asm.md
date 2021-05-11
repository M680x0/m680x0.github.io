---
title: "How does Clang handle inline assembly?"
layout: single
---
Recently I've been working on adding inline assembly support for our beloved M68k LLVM backend (and its friends). At the time of this writing the M68k target can only recognize simple -- if not trivial -- inline assembly statements. So I thought it might be a good idea to write down what I've done here and hopefuly become a general guide for new LLVM targets in the future.

## Inline Assembly in a Nutshell
Inline assembly is a feature that allows you to embed target-specific assembly code into normal C/C++ code. For instance:
```c
int foo() {
    asm volatile ("move.l #87, %%d0");
}
```
The `foo` function above contains a simple M68k assembly line that store constant value 87 into the `d0` register, which is essentially returning value 87 from function `foo`.

There are many reason people want to interoperate C/C++ and assembly code: Getting better run-time performance has always been a common reason. Or, you're doing some low-level system programming -- like writing an Operating System.

One of the motivations to add sturdy inline assembly support for M68k LLVM is the [ClangBuiltLinux](https://github.com/ClangBuiltLinux/linux) project. As you can guess from its name, this project has been exploring ways to build the _entire_ Linux kernel tree using Clang. And by saying _entire_ tree, M68k architecture is not an exception. Linux kernel is written in good'o C90 with GNU extensions (`-std=gnu89`). While Clang will take care of most C code, a none trivial amount of inline assembly code is completely left to the backend. Without support for these inline assembly statements, we cannot build the M68k Linux tree at all.

Inline assembly in Linux kernel -- or any serious real-world use case -- is usually more complicate than a single assembly string like the one we saw previously. For instance, developers can assign C/C++ variable as inputs/outputs to the assembly code:
```c
int foo(int x) {
    int y = x + 1;
    asm volatile ("move.l %0, %%d0" : : "r"(y));
}
```
The above code tries to replace the "%0" string with a register that stores the variable `y`. So if `y` is eventually put into `d1`, the assembly code will become `move.l %d1, %d0`. The `"r"(y)` part is essentially the input operand for this inline assembly statement.

In this blog post I'm going to cover three of these advance inline assembly features (or called [Extended Inline Assembly](https://gcc.gnu.org/onlinedocs/gcc/Extended-Asm.html) by GCC):
 - Input / output operand
 - Constraint
 - Operand modifier

Let's start with the big picture first.
## The Journey of an Inline Asssembly Statement