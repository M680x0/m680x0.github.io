---
title: "What's New in M68k LLVM (May 2023)"
date: 2023-05-24 22:00 -0700
author: mshockwave
---

It has been a minute[^1] since the last update on our [Open Collective](https://opencollective.com/m68k-llvm-dev) and [Patreon](https://www.patreon.com/m68k_llvm) campaigns.
So I thought it's a good idea to have a slightly more formal write-up on the progress we made in the past year.

[^1]: Last update was March 2022, my apology for the delay.

## Atomic Instructions
Atomic instructions are commonly seen in modern architectures to perform indivisible operations.
However, historically speaking, atomic instructions have never really been a thing for m68k, since processors in this family are predominantly single-core[^2], which is the model we primarily focus on in this project.
That said, as a backend we still need to lower atomic instructions passing from earlier stages in the compilation pipeline. Otherwise, LLVM will simply bail out with a crash.

[^2]: related: 68060 actually has superscalar support.

For atomic load and store, the stories are a lot simpler: due to the aforementioned single-core nature, lowering them to normal `MOV` instructions should be sufficient, which was something [D136525](https://reviews.llvm.org/D136525) did.
In the same patch, the author, [Sheng](https://github.com/0x59616e), also dealt with something more tricky: atomic compare-exchange (cmpxchg) and its friends, like atomic fetch-and-add (or add-and-fetch).
Despite being single-core, the processor can still run multi-tasking systems. So we need to make sure an atomic cmpxchg is immune to system routines like interrupts and/or context-switching.
To this end, 68020 and later processors are equipped with the [`CAS`](/ref/integer-instructions.html#pfaa) instruction, which can be used as the substrate for fetch-and-X instructions, in addition to implementing cmpxchg.
For older processors, we expanded these instructions into lock-free library calls (i.e. `__sync_val_compare_and_swap` and `__sync_fetch_*`).
In addition, this patch also lowered atomic read-modify-write (RMW) and any atomic operations larger than 32 bits into library calls of libatomic, which are not lock-free[^3].
Last but not the least, [85b37d0](https://reviews.llvm.org/rGa85b37d0ca819776c6034c2dbda2b21e54e3393a) added the lowering for atomic swap operations.

[^3]: LLVM has [an amazing page](https://llvm.org/docs/Atomics.html#atomics-and-codegen) documenting codegen for atomic operations, including the explanations to lock-free v.s. libatomic library calls. Highly recommended.

[D146996](https://reviews.llvm.org/D146996) was dealing with a similar puzzle: atomic fence.
As mentioned before, we don't need to worry about the memory operation order in a in-order single-core processor, like most members in 68k.
Thus, this patch only needs to prevent compiler optimizations from reodering instructions across atomic fence. I believe there is definitely a more sophisticate solution, like adding dependencies (e.g. SelectionDAG chains) between instructions placed before and after the fence...but, well, I was lazy so I literally copied what m68k GCC did: lower atomic fence into an _inline assembly_ memory barrier a.k.a `asm __volatile__ ("":::"memory")` (more precisely, an inline assembly instruction in LLVM's MachineIR).

That said, if we want to deal with potentially-out-of-order 68060 processors in the future, we might need to lower any fence into a `NOP`, which has the syntax of [synchronizing the pipeline](/ref/M68000PM_AD_Rev_1_Programmers_Reference_Manual_1992.html#pf5c).

## Floating Point Support
Similar to atomic instructions, another thing people might find surprised is the lack of (builtin) floating point insturctions in most 68k family processors.
Just like the original [x87](https://en.wikipedia.org/wiki/X87), m68k employed _co-processors_ for floating point (FP) operations, called 68881 and 68882 (after 68040, the 68881/2 are integrated into the main processor).
Luckily, compared to x87, m68k's FP instructions are much more straightforward. Notably, they use nearly identical addressing modes as their integer counterparts (except using floating point data registers, of course).
The list of FP instructions can be found [here](https://m680x0.github.io/ref/floating-point-instructions.html).

[D147479](https://reviews.llvm.org/D147479) and [D147481](https://reviews.llvm.org/D147481) laid down 68k's FP foundation like new register classes and compiler driver flags, in their respective LLVM and Clang components;
[D147480](https://reviews.llvm.org/D147480) and [D148255](https://reviews.llvm.org/D148255) added definitions and MC supports (AsmParser/Printer and disassembler) for an extremely limited number of data and arithmetic instructions.
Currently no codegen support has been added to these instructions, which is definitely one of our future plans.
Aside from codegen, an easier task might be adding preliminary inline assembly supports for floating point constraints and escaped characters, as described in [this issue](https://github.com/llvm/llvm-project/issues/61806).

## Aggregate-Type Return Values
One of the patches authored by our new contributor, [Ian](https://github.com/ids1024) (Welcome!🎉), was [D148856](https://reviews.llvm.org/D148856), which added supports for lowering aggregate-type return values like structs or arrays.

The way how a function returns aggregate values is heavily ABI-dependent. While many modern architectures leverage registers to return small structs (and use memory for larger ones), none of the 68k's ABIs specify such optimization.
Therefore, we simply return aggregate values by storing to a _caller-allocated_[^4] memory, whose pointer is passed from an implicit-inserted function argument.
So a C++ code like this:
```cpp
struct Hello {
  unsigned f1, f2;
  float f3;
};

Hello foo(unsigned v) {
  Hello obj{v, v, 8.7f};
  return obj;
}
```
will be translated to the following LLVM IR code when targeting m68k:
```
define void @_Z3fooj(ptr sret(%struct.Hello) %agg.result, i32 %v) {
entry:
  store i32 %v, ptr %agg.result
  %f2 = getelementptr inbounds %struct.Hello, ptr %agg.result, i32 0, i32 1
  store i32 %v, ptr %f2
  %f3 = getelementptr inbounds %struct.Hello, ptr %agg.result, i32 0, i32 2
  store float 0x4021666660000000, ptr %f3
  ret void
}
```
in which the `%agg.result` is the implicit-inserted argument used to return our aggregate value.

[^4]: SysV ABI actually didn't specify who (caller v.s. callee) should allocate the memory, but LLVM's codegen infrastructure designates it to caller by default. Plus, it makes more sense.

## Inline Assembly: Memory Constraints
We added the supports for most of the inline assembly constraints, either target [independent](https://gcc.gnu.org/onlinedocs/gcc/Simple-Constraints.html) or [dependent](https://gcc.gnu.org/onlinedocs/gcc/Machine-Constraints.html) ones about 2 years ago -- but memory constraints (e.g. `m`) had always been absent.

Wait no more! [D143529](https://reviews.llvm.org/D143529) just added the supports for `m`, `Q`, and `U` constraints.
The `m` constraint accounts for generic memory operands; `Q` is subject to any addressing modes that involves an address register as the base; `U` is similar to `Q`, but limited to those using constant offsets.

There were two challenges in this task: first, there are more than one possible addressing modes to which operands with any of those memory constraints can be lowered. We need to use instruction selector to select an addressing mode for the memory operand in question.
Unfortunately, in our backend, the selection logic for finding the optimal addressing modes is not easily accessible from outside the instruction selector, so we ended up trying every possible addressing modes, one after another, following an order I arbitrarily picked🤪.

Second, `AsmPrinter` -- the last Pass in the codegen pipeline -- is required to print the selected memory operand into the inline assembly string _without_ the help of MC[^5].
Since m68k's own MC component also has the exact same printing logics for memory operands, we ended up having duplicate code in two places (AsmPrinter and m68k's MC). To avoid that, [D143528](https://reviews.llvm.org/D143528) factored out the printing logics shared by so that both components can share.

[^5]: The reason being that, despite being rare, integrated assembler is not required for a target. Thus, we need to make sure those inline assembly memory operands are still properly printed in the absence of integrated assembler.

## Miscs

### Improvements on register spilling
[D133636](https://reviews.llvm.org/D133636) fixed the instructions emitted for register spilling.

### `TRAP` instruction and its friends
[D147102](https://reviews.llvm.org/D147102) added MC supports[^6] for `TRAP`, `TRAPV`, `BKPT`, `ILLEGAL`. These instructions are crucial for making (Linux) system calls.
This patch also added a special immediate operand class to verify the odd-sized 3-and 4-bit immediate values used in some of these instructions.

[^6]: It's almost exclusively used in inline assembly so no codegen support is needed.

### Some exception handling supports
[058f744](https://reviews.llvm.org/rG058f7449cf38fcbfff4a54a1f67a784ca2983671) specified the registers for exception handling: `d0` for exception pointer and `d1` for selector, as suggested by GCC.

### Better `-stop-before/after` flags
Started by [Nick](https://github.com/nickdesaulniers) in [D140364](https://reviews.llvm.org/D140364) and followed by [3204740](https://reviews.llvm.org/rG3204740bde796fc37251acddb7b3e123c0ec9196), most of the machine Passes in 68k's codegen pipeline are now registered with sane names. So that it's easier for backend developers to stop/(re)start from a specific point in the codegen pipeline with the `-stop-before/after=<pass name>` flags.

## // End Report