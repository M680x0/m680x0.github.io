---
title:  "Code Generation for M68k Floating Point Instructions"
author: mshockwave
layout: single
classes: wide
---

While many of the 68k floating point instructions have been implemented in the LLVM backend for a while, their code generation (codegen) support -- translating from LLVM IR to
these native instructions -- are still absent. Meaning that floating point operations in the programs built by our compiler cannot leverage these hardware features[^1] but fallback to (much) slower software emulations.

[^1]: Assuming M68881/2 co-processors are available or the CPU is M68040 or later.

In this post, I'm going to outline the design of floating point codegen support in our 68k LLVM backend and talk through the main challenges we have.

## Floating point data types and conversion
As a starter, let's talk about how floating point data is represented and modeled in 68k.
The FP unit in 68k supports 4 different floating point data types, of which we're only going to cover 3 of them:
  - 32-bit **S**ingle-precision Real (F32)  
  - 64-bit **D**ouble-precision Real (F64)
  - 80-bit e**X**tended-precision[^2] Real (F80)


All three conform to the IEEE-754 standard we're familiar with...well probably not so familiar with F80, as it was originally coming from Intel X87 (which has the same format we use here) and later ratified into IEEE-754.

[^2]: It's called _double-extended_ format in IEEE-754.

Now, normally when an ISA supports multiple different floating point types, which is pretty common, it creates an instrution for each data type.
For instance, in RISC-V, floating point ADD has two variants: `fadd.s` for single-precision (32-bit) and `fadd.d` for double-precision (64-bit). Each of these instructions interprets its inputs and result with the _same_ data type. If we want to convert a value from one type to another, there are dedicated instructions for this (e.g. `fcvt.s.d` in RISC-V). Straight forward, right?

The thing is, floating point instructions in M68k don't do this way -- nearly every floating point operations can have implicit data conversion.

Take `fadd.l %d2, %fp1` as an example, it adds a long word (32-bit) integer from `%d2` with the value stored in FP data register `%fp1` and puts result in `%fp1`. The ".l" suffix in the opcode tells us the _source_ data format, which is long word in this case.
For the _destination_, we have to specify its format via floating point control register, `%fpcr`. More specifically, the rounding precision (PREC) field.
Single, double, and extended precision we introduced earlier are the possible values in this field.
<br/>
<img src="/assets/m68k-fpcr-layout.png" style="margin: 30px;" width="70%"> 
<br/>
The value in `%d2` will be converted into an extended precision floating point value before doing the summation. After that, it might need to do another conversion depends on the
rounding precision we choose in FPCR.

It is true that in the worst case, there will be _two_ data conversions happening in a single arithmetic instruction and that looks scary. But the real challenges for code
generation are the facts that:
  1. We can use source data of nearly **every** data types supported in M68k! Including all three 8/16/32-bit integers and all floating point formats
  2. Floating point precision of the destination is implicitly specified through FPCR rather than being part of the instruction opcode
  3. Similar to (2), the _rounding mode_ used for conversion is also implicitly specified through FPCR.

These challenges add the difficulty of how we model an FP instruction in both code generation stage and assembler level (i.e. MC layer in LLVM). Naively, we can just stuff all three factors -- source data type, destination data type, and rounding mode -- into the opcode of an instruction during code generation. For instance, `FADD_f32_i32_rtn` can be the opcode of an instruction that adds a 32-bit integer to a single precision floating point value, using round-to-nearest rounding mode.

However, since both floating point precision and rounding mode are not explicitly encoded into the final assembly instruction, opcode `FADD_f32_i32_rtn` and `FADD_f64_i32_rtz` will actually be lowered into the _same_ assembly instruction despite being different opcodes during code generation.
Sadly, LLVM currently doesn't allow two instructions to share the same encoding.

## Solution 1: Marking instructions as "codegen only"
LLVM provides a leeway for multiple instructions to share the same encoding: marking them as "codegen only", via the `isCodeGenOnly` field in an instruction's TableGen definition. For instance, with instructions A, B, and C sharing the same encoding, we mark A and B as codegen only such that the assembler will treat A and B as C whenever possible.

Of course, once codegen only instructions are converged we'll loose part of their numerical information, so we have to make sure the precision and rounding mode fields in FPCR are updated accordingly through all floating point instructions before sending them into the assembler.

Now, let's do a simple math on how _many_ opcodes does this solution produce for each instruction. First, we support a total of 3 integer types and 3 floating point types; we can put these 6 possible
types on either source or destination (but not both, since something like integer-to-integer operation doesn't make sense), and 3 floating point types on the other; there are a total of 4 rounding modes.
So each FP instruction might have 6 x 3 x 4 = _72_ different opcodes!

Hold on, something is still missing: _addressing modes_.
Remember 68k has a total of _15_ different memory addressing modes? Luckily our LLVM compiler currently only supports roughly 10 of them, and don't worry there is no memory-to-memory
floating point operations in 68k -- but that still gives us a whooping **720** different opcodes for **each** instruction!

## Solution 2: Pseudo instructions to the rescue
Apparently, solution 1 doesn't scale well. A common solution to this kind of problem is "offloading" some properties from opcode space to instruction operands.
For instance, in our case, we can use an immediate operand to carry rounding mode information. So instead of using `FADD_f32_i32_rtn` we have:
```
// Value 1 means RTN
%r = FADD_f32_i32   %0, %1, /*rounding mode = rtn*/ 1
```
That is, the last operand carries the rounding mode for this instruction.

However, rounding mode information is not going to be encoded into our final assembly instructions. Meaning, with this approach instructions during codegen has a different form than its assembly instruction.

This is where pseudo instruction usually comes into place: pseudo instructions are instructions without any encoding information, therefore you have to convert it into another instruction(s) at later
stages, if not the very end, of codegen pipeline. Since you have to do the conversion anyway, people usually carry additional information that is only used during codegen as extra operands.
Which is basically what we did in the `FADD_f32_i32` example earlier.

By turning opcodes like `FADD_f32_i32_rtn` into `FADD_f32_i32`, we slash the number of opcode needed per instruction by 4x, yielding around 180 opcodes per instruction.

Nice! but can we do more?

Unfortunately, this is probably the best we have in terms of number of opcodes. Because the other three factors we mentioned earlier -- source / destination data type, and addressing mode -- are part of the _operand list_ signature. Each opcode can only have a single operand signature, so we need to pay the same number of opcodes for every possible operand signatures, consistuted by different combinations of, you guess, source / destination data type and addressing mode.

## Solution 2.5 (final solution): Limiting operand types
TL;DR: Data conversions are rare.