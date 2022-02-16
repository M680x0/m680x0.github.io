---
title:  "Encoding Variable-Length Instructions in LLVM"
date:   2022-02-15 17:00 -0800
author: mshockwave
---

One of the most important jobs for an assembler is encoding assembly instructions -- potentially in a textual form -- into their corresponding binary code, namely the instruction encoding. LLVM has provided a nice framework to spare toolchain developers from crafting this process manually, by _generating_ such encoder, also called code emitter, from high-level target instruction descriptions. That is, the instruction info written in TabelGen. This feature greatly improves the productivity and makes the target-specific codebase more readable.

However, such framework came with an assumption that is unfavorable to our M68k target: _All target instructions must have the same size_. Where M68k has always been an ISA with variable-length instructions, ranging from 2 bytes to 22 bytes, since day one.

In this blog post, I'm going to talk about how M68k LLVM overcomes this issue by augmenting the existing code emitter generator with more powerful TableGen syntax.

First, let's talk about some backgrounds and history behind M68k's instruction encoding scheme.

## The original code emitter generator (CodeEmitterGen)
The core idea of the original code emitter generator is to generate the instruction encoder's C++ code from (target-specific) TableGen instruction definitions via a special TableGen backend, namely the CodeEmitterGen.

For instance, assuming we have a hypothetical instruction, `MyInst`, that has the following encoding scheme:
```
  15       12  11         8   7             0
| 1  0  1  1 | dst register | 8-bit immediate |
```
In this scheme, bit 15 to 12 are fixed values. Bit 11 to 8 and 7 to 0, on the other hand, contain the encoded value of destination register and a 8-bit immediate, respectively. So for instance, if the destination register is `r8`, which is encoded as `0b1001` in our hypothetical architecture, and `0b01010111` for the immediate, then this `MyInst` will eventually be encoded as
```
1011_1001_01010111
```

The key is that an instruction is usually consisting of fixed values -- in this case `0b1011` on bit 15 to 12 -- and _dynamic_ parts -- dst register and 8-bit immediate -- that varies among different instances. Which are usually the instruction operands.

The TableGen instruction definition also accounts for this fact. Below you can see the definition for `MyInst`:
```javascript
class MyInst : Instruction {
    let OutOperandList = (outs GR64:$dst);
    let InOperandList  = (ins GR64, i8imm:$imm);

    bits<16> Inst;

    bits<4> dst;
    bits<8> imm;
    let Inst{15-12} = 0b1011;
    let Inst{11-8} = dst;
    let Inst{7-0} = imm;
}
```
We can ignore the `OutOperandList` and `InOperandList` fields for now. The `Inst` field is where we put the instruction encoding scheme.

For the fixed values part, we simply assigns the bits to the corresponding bit segment:
```javascript
let Inst{15-12} = 0b1011;
```
For the dynamic parts, it is twofold: First, we declare a new field (i.e. `bits<4> dst` and `bits<8> imm`) for each instruction operand. Then, these uninitialized fields are assigned to their corresponding bit segments right away:
```javascript
let Inst{11-8} = dst;
let Inst{7-0} = imm;
```
The CodeEmitterGen TableGen backend will then interpret these descriptions and generate an instruction encoder.

<br>

Alright, so this is the status quo of how majority of the LLVM targets (which have fixed-length instructions) describe their instruction encoding schemes. Let's try to adopt this framework in our M68k target.

Right off the bat, we need to solve the problem that the `Inst` field has a fixed size. The most intuitive solution is, of course, using the maximum instruction size of the ISA -- 22 bytes in M68k's case. But then for every encoded instructions emitted from the encoder, it will always be 22 bytes long regardless of it's instruction type, which is not what we want, since CodeEmitterGen was designed with fixed-length instruction in mind.

Luckily, there is a post-processing callback we can define to massage the size of encoded instructions:
```javascript
// MxInst is the base class for all the M68k instructions.
class MxInst : Instruction {
    let PostEncoderMethod = "adjustInstSize";
    ...
}
```
By specifying the name of the post-processing function in the `PostEncoderMethod` field, the generated encoder will use that function to give the "raw" encoded instruction -- in this case the 22-byte long instruction -- a pass before presenting the final result. In this case, the `adjustInstSize` function might look like this:
```c++
APInt M68kMCCodeEmitter::adjustInstSize(const MCInst &MI, const APInt &RawValue,
                                        const MCSubtargetInfo &STI) {
    unsigned RealSizeInBits = 22 * 8;
    switch (MI.getOpcode()) {
        case M68k::ADD16ri:
        case M68k::SUB16ri:
        ...
            RealSizeInBits = 32;
            break;
        ...
    }

    return RawValue.truncOrSelf(RealSizeInBits);
}
```
Basically, we copy the original encoded instruction and truncate to its real size, deduced from its opcode. Though this also means we need to enumerate all of opcodes in this function, it's not a big deal. Actually, later on we will see that the problem we just discussed is the _least_ tricky one.

Next on our issue list, we have M68k instructions whose _operands_ might vary in their sizes. Take the `move.w` instruction below as an example:
```
move.w %d0, (87,%a1)
```
Which moves data from `%d0` to `(87,%a1)`, the destination. And its encoding has a layout like this:
```
 31                               16 15                                0
|  immediate in destination operand |          Base encoding            |
```
Alternatively, we can use an immediate value as the source operand:
```
move.w #94, (87,%a1)
```
In this case, the instruction size increases to 48 bits, with the following layout:
```
 47                               32 31                        16 15                                0
|  immediate in destination operand | source operand (immediate) |          Base encoding            |
```
The problem here is that a single instruction type can have dramatically different operands in terms of their sizes and bit positions (immediate in the destination operand was at 31 \~ 16 in the first case, but moved to 47 \~ 32 in the second one). But in the framework provided by CodeEmitterGen, we can only assign an operand's placeholder field into a fixed position (e.g. `let Inst{11-8} = dst` we saw earlier).

A potential solution will be creating an opcode for each of these instances. So in our previous example, `move.w %d0, (87,%a1)` and `move.w #94, (87,%a1)` will have different opcodes. This is not a bad solution, to be honest. In fact, most of the LLVM targets use this very approach when encountering similar cases.

The real problem is, it doesn't _scale_ in M68k.

In M68k, there are at least 12 different types -- or what you can call **addressing modes** -- of operands. And each M68k instruction, unfortunately, supports nearly _all_ of them. That means, for all 190+ differnt instruction types, we need to compose -- 12 times 190 -- around 2000 different instruction definitions in our TableGen file!

You may ask: "Wasn't TableGen invented to eliminate such repetition?". Well, TableGen does have amazing features to factor out common patterns...except for those we've seen here. More specifically, the fact that an operand's placeholder field can only be assigned to a fixed position, which we discussed previously, really prevent us from factoring out anything useful.

> We can, of course, change the TableGen language and
> allow non-literal values for indicies in `bits` slicing
> (e.g. `let Inst{8-n} = ...` where `n` is a template variable).
> But it seems to be much more difficult if not impossible.

Last but not the least issue, we have complicated M68k instruction operands that are not even located at contiguous bit positions! In the previous issue we saw this instruction layout for `move.w %d0, (87,%a1)`:
```
 31                               16 15                                0
|  immediate in destination operand |          Base encoding            |
```
The reason I said "immediate in destination operand" rather than "destination operand" was because bit 31 \~ 16 was just _part_ of the destination operand, namely the 87 part in `(87,%a1)`. For the other part, base register `%a1`, it is located at bit 2 \~ 0.

However, CodeEmitterGen's TableGen framework doesn't provide any mechanism to assign such sub-operands into `Inst`. A potential workaround will be assigning the operand's placeholder to every sub-operand positions. For example:
```javascript
let Inst{31-16} = dst;
let Inst{2-0} = dst;
```
But then we need a way to assigned the correct encoded value to the right bit position.

LLVM calls into a callback function, which can be customized on a per-operand basis, whenever it needs to encode an operand. It has the following function signature:
```c++
uint64_t getMachineOpValue(const MCInst &MI, const MCOperand &MCO,
                           SmallVectorImpl<MCFixup> &Fixups,
                           const MCSubtargetInfo &STI);
```
or the following when instruction size exceeds 64 bits:
```c++
void getMachineOpValue(const MCInst &MI, unsigned OpIdx, APInt &Result,
                       SmallVectorImpl<MCFixup> &Fixups,
                       const MCSubtargetInfo &STI);
```
In the previous snippet, since we assign `dst` to both sub-operands' positions, the same callback function will be invoked twice on those positions -- with the same arguments! (Note that both `MCO` and `OpIdx` provide an operand rather than a sub-operand) In other words, a callback has no way to tell what _sub-operand_ it is encoding now.

<br>

In summary, in the original CodeEmitterGen, we observe the following shortcomings that are unfavorable to variable-length instructions:
  1. Bit width of the `Inst` field is fixed. Though we can declare the field with maximum instruction size in the ISA, it requires extra code to adjust the final instruction size.
  2. Operand encoding can only be placed at fixed bit positions. However, the size of an operand in a variable-length instruction might vary.
  3. In the situation where a single logical operand is consisting of multiple sub-operands, the current syntax cannot reference a sub-operand. Which means we can only reference the entire logical operand at places where we actually should put sub-operands. Making the TG code less readable and bring more burden to the operand encoding functions (because they don't know which sub-operand to encode).

## M68k's old instruction encoder
We've spent some time exploring some options of adopting the original CodeEmitterGen (and why it didn't work out). I think it's also worth it to take a look at M68k LLVM's previous instruction encoder: **CodeBeadsGen**.

CodeBeadsGen is a TableGen backend created by [Artyom Honcharov](https://github.com/m4yers). On the face of it, CodeBeadsGen simply converts a `bits` type TableGen field, `Beads`, in each instruction definition into an `uint8_t` array. In M68k, we used this stream of bits as the vehicle for instruction encoding fragments that are concatenated one after another, thus got the name code _beads_. Each encoding fragment can represent either a fixed bits value or placeholders for operands -- similar to the original CodeEmitterGen except that fragments are organized as a sequence.

For example, `class MxBead4Bits<bits<4> value>` is a fragment representing a 4-bit wide fixed value passing from the template argument `value`; `class MxBeadDReg<bits<3> op_idx>` is a fragment for data register, where `op_idx` is the index of the corresponding operand; similarly, `class MxBead8Imm<bits<3> op_idx>` is a fragment for 8-bit immediate, and `op_idx` is also the index of the corresponding operand. 

In addition to fragments, an utility class, `MxEncoding`, helps us to convert a list of fragments into raw `bits`. Here is how they work together:
```javascript
let Beads = MxEncoding<MxBead8Imm<2>,
                       MxBeadDReg<1>,
                       MxBead4Bits<0b0100>>.Value;
```
The `Beads` above represents the following encoding layout:
```
  15    12  11                      8   7                         0
| 0 1 0 0 | data register (operand 1) | 8-bit immediate (operand 2) |
```
Note that each fragment still requires further interpretation (by the `M68kMCCodeEmitter` class) to emit the real encoded value.

The reason I mention this legacy framework here is because there are two things I want to highlight:
  1. Fragment are position-independent. As a developer, we only need to put fragments in the correct ordering. The exact bit positions will be handled by the framework.
  2. We can reuse fragments, or even compose a new one by ourselves. For example, we can create a new fragment for the `(87,%a1)` addressing mode we saw earlier by putting two fragments, `MxBead16Imm` and `MxBeadAReg` (for address register), together. Such aggregate fragment can also be reused multiple times in other parts of the codebase.

Some core designs in the new CodeEmitterGen, which we will cover shortly, are actually heavily influenced by these two concepts.

<br>

So why did we drop this CodeBeadsGen-based instruction encoder? There are primarily two reasons:
  1. An operand fragment (e.g. `MxBeadDReg`) uses operand index rather than more ergonomic mnemonic to designate an operand. What's worse, the _index_ we're talking here is actually not the logical operand index, but the `MCOpreand` index. A complex logical operand like `(87,%a1)` is actually consisting of multiple `MCOperand`-s. And it's pretty tricky to figure out one of their index, because you need to account for the number of `MCOpreand`-s before this logical operand.
  2. The fact that every fragments are converted into raw bits before being consumed by `M68kMCCodeEmitter` forcing us to funnel lots of metadata and annotations into those bits. But then in `M68kMCCodeEmitter`, we need to spend even more energies to decode those metadata! This eventually created many obscured TableGen and `M68kMCCodeEmitter` codes / logics, which are hard to maintain.

## Introducing the new VarLenCodeEmitter
Finally, let's talk about **VarLenCodeEmitter** -- a new extension to CodeEmitterGen that adds supports for encoding variable-length instructions.

Following up our previous `MyInst` hypothetical instruction, let take a look one of its more advanced siblings: `MyVarInst`.
```javascript
class MyMemOperand<dag sub_ops> : Operand<iPTR> {
    let MIOperandInfo = sub_ops;
}

class MyVarInst<MyMemOperand memory_op> : Instruction {
    let OutOperandList = (outs GR64:$dst);
    let InOperandList  = (ins memory_op:$src);
}
```
The `OutOperandList` and `InOperandList` are fields for instruction's input and output operands, respectively. In here, we have a slightly more complex input operand type, `MyMemOperand`, which might contain more than one sub-operand. For example, `MemOp16` and `MemOp32` below are both consisting of two sub-operands.
```javascript
def MemOp16 : MyMemOperand<(ops GR64:$reg, i16imm:$offset)>;
def MemOp32 : MyMemOperand<(ops GR64:$reg, i32imm:$offset)>;
```
Notice that both `memory_op` (in `InOperandList`) and sub-operands within `MyMemOperand` are tagged with names like `$src`, `$reg`, or `$offset`.

Now, we know `MyVarInst` has the following instruction encoding:
```
15             8                                   0
----------------------------------------------------
|   10110111   |  Sub-operand 0 in source operand  |
----------------------------------------------------
X                                                 16
----------------------------------------------------
|         Sub-operand 1 in source operand          |
----------------------------------------------------
                X + 4                          X + 1
                ------------------------------------
                |       Destination register       |
                ------------------------------------
```
We put many `X`-s in the diagram above because the size of a sub-operand might vary.
Similar to the original CodeEmitterGen, we're also storing encoding info in the `Inst` field -- except that it is `dag` type rather than `bits`. Here is an example:
```javascript
class MyVarInst<MyMemOperand memory_op> : Instruction {
    let OutOperandList = (outs GR64:$dst);
    let InOperandList  = (ins memory_op:$src);

    dag Inst = (ascend
        (descend /*Fixed bits*/0b10110111,
                 /*Sub-operand 0 in source operand*/(operand "$src.reg", 8)),
        // Sub-operand 1 in source operand
        (operand "$src.offset", 16),
        // Destination register
        (operand "$dst", 4)
    );
}
```
The idea is to use special dag operators and TableGen values (e.g. fixed bits `0b10110111`) to build a _sequence_ that reflects the encoding format. For instance, the following snippet represents the encoding from bit 15 \~ 0:
```javascript
(descend /*Fixed bits*/0b10110111,
         /*Sub-operand 0 in source operand*/(operand "$src.reg", 8))
```
And for this snippet
```javascript
// Sub-operand 1 in source operand
(operand "$src.offset", 16)
```
It represents the encoding from bit X \~ 16.

Instead of specifying the exact bit positions where each part is going, we're merely assembling these parts with the correct _relative_ ordering.

Here are the descriptions of each dag directive we just saw:
  - `(ascend [value1, value2, ...])`: DAG arguments (i.e. `value1`, `value2`, ...) are concatenated one after another to form a sequence. They are organized from least-significant bit (LSB) to most-significant bit (MSB). That is, `value1` will sit at lower bits and `value2` will sit at higher bits. Each argument can be a `bits<N>` or another dag. 
  - `(descend [value1, value2, ...])`: Similar to `ascend`, but the arguments are orgnized from MSB to LSB.
  - `(operand <operand reference string>, <size in bits>)`: Representing the placeholder for an instruction operand. The first argument is a string that references the name -- for example, `$dst` or `$src` we saw earlier -- tagged on an operand. If there are sub-operands, we can use `$<operand name>.<sub-operand name>` to reference the sub-operand. The second argument specifies the size of the encoded operand.

Using `ascend` and `descend` together can help developers to transcribe instruction format descriptions from the architecture manual. Since many of them write MSB to LSB from left to right. But if there is a need to split the encoded bits into multiple "rows" (e.g. a row is a word), the top row is sitting at lower bit while the bottom is sitting at higher bit. For example:
![stack instruction description](/assets/img_1cb2db7e1b0bf80faf9b3b9e399f70031b0d38b5.png)
Where _"BASE DISPLACEMENT"_ is sitting at bit 16 \~ 31 and _"OUTER DISPLACEMENT"_ sits at bit 63 \~ 32. In this case, using the following syntax will be easier to understand:
```javascript
let Inst = (ascend
  (descend ...), // bit 15 ~ 0
  (descend ...), // bit 31 ~ 16
  (descend ...)  // bit 63 ~ 32
);
```

<br>

Now let's pause for a second and look into the example snippet we just went through:
```javascript
let InOperandList  = (ins memory_op:$src);

dag Inst = (ascend
    (descend /*Fixed bits*/0b10110111,
              /*Sub-operand 0 in source operand*/(operand "$src.reg", 8)),
    // Sub-operand 1 in source operand
    (operand "$src.offset", 16),
    // Destination register
    (operand "$dst", 4)
);
```
In the above, sub-operand 0 and 1 in the source operand are explicitly referenced by `(operand "$src.reg", 8)` and `(operand "$src.offset", 16)`, respectively.

But what if...
  - We have a source operand (i.e. `memory_op`) whose `offset` sub-operand has a size different than 16 bits? A good example is the `MemOp32` we created earlier.
  - Instead of "reg" and "offset", `memory_op` has different names for its sub-operands.

To address these questions, let's make our code more flexible. First, let's change the class definition of `MyMemOperand`
```javascript
class MyMemOperand<dag sub_ops> : Operand<iPTR> {
    let MIOperandInfo = sub_ops;
    dag Base;
    dag Extension;
}
```
After finishing that, we change the content of `Inst` in `MyVarInst`:
```javascript
dag Inst = (ascend
    (descend /*Fixed bits*/0b10110111,
              /*Sub-operand 0 in source operand*/memory_op.Base),
    // Sub-operand 1 in source operand
    memory_op.Extension,
    // Destination register
    (operand "$dst", 4)
);
```

<br>

Now let's look at the operand. For `MemOp16`, instead of creating a TableGen record from `MyMemOperand`, we create a class first:
```javascript
class MemOp16<string op_name> : MyMemOperand<(ops GR64:$reg, i16imm:$offset)> {
    let Base = (operand "$"#op_name#".reg", 8);
    let Extension = (operand "$"#op_name#".offset", 16);
}
```
> The `"$"#op_name#".reg"` syntax concatenates three strings -- "$", op_name, and ".reg" -- together.

Similarly...
```javascript
class MemOp32<string op_name> : MyMemOperand<(ops GR64:$reg, i32imm:$offset)> {
    let Base = (operand "$"#op_name#".reg", 8);
    let Extension = (operand "$"#op_name#".offset", 32);
}
```
Finally, we instantiate new `MyVarInst` instances like this:
```javascript
def FOO16 : MyVarInst<MemOp16<"src">>;
def FOO32 : MyVarInst<MemOp32<"src">>;
```

<br>

What we just did is factoring out the parts that are related to operands. This not only makes `MyVarInst` more generic, the new `MemOp16` and `MemOp32` classes become _composable_ and _reusable_ modules where we can use in other instruction definitions:
```javascript
class My2ndVarInst<MyMemOperand memory_op> : Instruction {
    ...
    let InOperandList  = (ins memory_op:$opnd);
    ...
}

def BAR16 : My2ndVarInst<MemOp16<"opnd">>;
```
We don't need to repeat operand encoding in every instruction definitions anymore.

<br>

To sum up the advantages of using VarLenCodeEmitter:
  1. Developers can express instruction encodings in a more ergonomically way and avoid details like exact bit positions, which is a problem for variable-length instruction. Making instruction definitions much easier to read and maintain.
  2. It's easier to create composable components, which facilitates code reuse.
  3. VarLenCodeEmitter shares many traits and most of the interfaces with the original CodeEmitterGen. For example, both of them generate `<Target>MCCodeEmitter::getBinaryCodeForInstr` as the primary encoder (function body is of course different but function signature is the same).
  4. It's a generic framework that is not limited to variable-length instructions -- you can use it on normal fixed-length instructions too!

### Adopting VarLenCodeEmitter
To adopt VarLenCodeEmitter, in addition to using the TableGen syntax introduced ealier, it shares the same `llvm-tblgen` flag as the original CodeEmitterGen:
```
# In your target's CMakeLists.txt
tablegen(LLVM <Target>GenMCCodeEmitter.inc   -gen-emitter)
```
Namely, after you write a `dag`-type `Inst` field (rather than `bits`) in your instruction definitions, VarLenCodeEmitter will automatically take over.

#### _Thank you for reading!_

## Appendix
There are two other useful dag constructions in VarLenCodeEmitter:
  - `(slice <operand reference string>, <starting or ending bit>, <starting or ending bit>)`: Similar to `operand`, but instead of referencing the entire operand, we're referencing part of the operand encoding, bounded by second and third dag arguments.
  - `(operand ..., (encoder <encoder function name>))` and `(slice ..., (encoder <encoder function name>))`: The `encoder` is an extension to both `operand` and `slice`. It specifies a custom C++ encoder function for this specific operand encoding, rather than the default `getMachineOp`. This is similar to `EncoderMethod` in an `Operand` TableGen record, but `EncoderMethod` applies to _every_ operands in the current target whereas `encoder` here only affects the enclosing `operand` or `slice`.

> **NOTE**
> This article is adapted from my [previous write up on Gist](https://gist.github.com/mshockwave/66e98d099256deefc062633909bb7b5b).