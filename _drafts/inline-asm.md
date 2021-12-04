---
title: "How does Clang handle inline assembly?"
layout: single
---
Recently I've been working on adding inline assembly support for our beloved M68k LLVM backend (and its friends). Before writing this, the M68k target can only recognize simple -- if not trivial -- inline assembly statements. So I thought it might be a good idea to write down what I've done here and hopefuly become a general guide for new LLVM targets in the future.

## Inline assembly in a nutshell
Inline assembly is a feature that allows you to embed target-specific assembly code into normal C/C++ code. For instance:
```c
int foo() {
    asm ("move.l #87, %%d0" ::);
}
```
The `foo` function above contains a simple M68k assembly line that store constant value 87 into the `d0` register, which is essentially returning value 87 from function `foo`.

There are many reason people want to interoperate C/C++ and assembly code: Getting better run-time performance has always been a common reason. Or, you're doing some low-level system programming -- like writing an Operating System.

One of the motivations to add sturdy inline assembly support for M68k LLVM is the [ClangBuiltLinux](https://github.com/ClangBuiltLinux/linux) project. As you can guess from its name, this project has been exploring ways to build the _entire_ Linux kernel tree using Clang. And by saying _entire_ tree, M68k architecture is not an exception. Linux kernel is written in good'o C90 with GNU extensions (`-std=gnu89`). While Clang will take care of most C code, a none trivial amount of inline assembly code is completely left to the backend. Without support for these inline assembly statements, we cannot build the M68k Linux tree at all.

Inline assembly in Linux kernel -- or any serious real-world use case -- is usually more complicate than a single assembly string like the one we saw previously. For instance, developers can assign C/C++ variable as inputs/outputs to the assembly code:
```c
int foo(int x) {
    int y = x + 1;
    asm ("move.l %0, %%d0" : : "r"(y));
}
```
The above code tries to replace the "%0" string with a register that stores the variable `y`. So if `y` is eventually put into `d1`, the assembly code will become `move.l %d1, %d0`. The `"r"(y)` part is essentially the input operand for this inline assembly statement.

In this blog post I'm going to cover three of these advance inline assembly features (or called [Extended Inline Assembly](https://gcc.gnu.org/onlinedocs/gcc/Extended-Asm.html) by GCC):
 - Operand and constraint
 - Operand modifier
 - Clobber

Let's start with the big picture first.

## Journey of an inline ASM inside the compilation pipeline
Before going into details of each topic above, let me give you a quick tour on how is an inline assembly statement being processed by Clang and LLVM.

I'm using the first snippet we saw previously as the driven example:
```c
int foo() {
    asm ("move.l #87, %%d0" ::);
}
```
The above code will be parsed into the following LLVM IR:
```llvm
define i32 @foo() {
entry:
  %retval = alloca i32, align 4
  call void asm sideeffect "move.l #87, %d0", ""()
  %0 = load i32, i32* %retval, align 4
  ret i32 %0
}
```
As you can see, LLVM has dedicated IR instruction -- `asm` -- for inline assembly. It leaves most of the assembly string content untouched (except input/output operand directives and some constraints, but we'll get there later).

Not only does LLVM IR preserve the assembly string, Machine IR (MIR) also uses a special instruction, `INLINEASM`, to model it. Here is the MIR of function `foo`:
```bash
$ llc -mtriple=m68k -stop-after=machineverifier input.ll -o -
...
body:             |
  bb.0.entry:
    liveins: $a6

    MOV32er $sp, killed $a6, implicit-def $ccr
    $a6 = frame-setup MOV32aa $sp, implicit-def $ccr
    INLINEASM &"move.l #87, %d0", 1 /* sideeffect attdialect */, !3
    $a6 = MOV32ro $sp, implicit-def $ccr
    RTS
$
```
Now a slightly interesting thing happens when the above MIR code is gradually lowered into native assembly code at the very end of code generation. If your LLVM target does _not_ have an integrated assembler -- which means that the code generation will end up emitting an assembly file -- the inline assembly string will simply be copied-and-pasted into the output file. But for LLVM targets that have an integrated assembler, including M68k, the inline assembly string will be parsed by the target-specific `AsmParser` before sending into the integrated assembler.

> **Integrated assembler** is nothing fancy but a normal assembler that is developed inside LLVM's source tree. Since integrated assembler, like other LLVM components, is just a library, the biggest advantage of having an integrated assembler for your LLVM target is that running the assembling process is merely calling an API. In contrast to invoking a command line tool when using _external_ assembler, like GNU AS.

More specifically, an `AsmParser` will parse the inline assembly string into `MCInst` instances, before combing with other `MCInst` compiled from "normal" code and sending to the integrated assembler.

> `MCInst` is the abstract representation of "a native instruction". The integrated assembler always takes a stream of `MCInst` as input and transforms them into either textual assembly code or binary object code.

If you're developing a LLVM backend -- regardless of having integrated assembler or not -- the above workflow will just kick in when the required codegen components are finished. So you don't need extra steps to support inline assembly...**trivial** inline assembly.
Unfortunately if you need to support other functionalities like operands and constraints, we need to make some changes on various compilation stages, from Clang to LLVM codegen. In the next section I'm going to talk about inline assembly operand, constraint, and how to implement them.

## Operand and operand constraint
Previously we knew that inline assembly statement can use C variables (and values) as input or output. For instance:
```c
int foo(int x) {
    int y = x + 1;
    asm ("move.l %0, %%d0" : : "r"(y));
}
```
In this snippet, variable `y` is the input to the inline assembly statement.

More formally speaking, the two colons follow after the assembly string is used to specified inputs and outputs to this statement.
```c
asm ("assembly string" : /*output operands*/ : /*intput operands*/)
```
There can be multiple intput/output operands in the spaces splitted by colons. Each operand has the following format:
```c
"constraint letters"(/*C expression*/)
```
Take the snippet at the beginning of this section, _"r"_ is the constraint letter and `y` is the C expression.
These operands will eventually replace any sub-string with the format "%\<_number_\>" within the inline assembly string. In which `number` is the index of the corresponding input or output operand.

Now we know the role of C expression here, but what is a **constraint**?

We're always taught to not making any assumption of whether a C/C++ variable (or value) will eventually be put on the stack or in an register when it's compiled into machine code -- because either one might happen. It's compiler's job to make this decision. However, when we're dealing with inline assembly, we _need_ to care about the location of an input/output C/C++ variable. Because as we introduced in the previous section, compiler will not modify the inline assembly string at all, so if we don't _control_ the final location of an input or output operand, we might create invalid asssembly syntax. 

In addition to the constraint letter, we can also decorate constraint with **modifiers** (a.k.a constraint modifier, please don't get confused with _operand_ modifier). For instance, output operands always need to start with a '=' modifier (for writing) or a '+' modifier (for reading / writing).
```c
void foo() {
    int out;
    asm ("move.l %%d0, %0" : "=r"(out):);
}
```
We're not going into the details of constraint modifier in this series of posts though.

We can roughly categorize constraints into three different kinds: Register, immediate, and memory.