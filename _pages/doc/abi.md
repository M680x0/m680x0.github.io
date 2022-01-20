---
title: "M68k Application Binary Interface (ABI)"
toc: true
---
## Variants of ABIs
In this document, we're covering two variants of M68k ABIs:
  1. **Linux/GCC ABI**: The default ABI used by GCC on Linux. It is backward-compatible with many legacy compilers' ABIs, such as Sun [PCC](https://en.wikipedia.org/wiki/Portable_C_Compiler). Yet it lacks formal documentations. This is currently the only ABI implemented by M68k LLVM.
  2. **SystemV ABI**: The default ABI used by UNIX systems. There is a nice [book](/ref/sysv-abi-download) documenting related details. Most of the contents in this page are also adapted from it.

## Data Representations

### Scalar Types
Here are the sizes and alignments of all scalar types.

<style>
    th, td {
        border: 1px solid;
    }

    th {
        text-align: center;
    }

    tbody td:nth-last-child(-n+3) {
        text-align: center;
        vertical-align: middle;
    }
</style>
<table>
    <thead>
        <tr>
            <th rowspan="2">Type</th>
            <th rowspan="2">C Type</th>
            <th rowspan="2">sizeof</th>
            <th colspan="2" style="border-bottom: 1px solid;">
                Alignment (bytes)
            </th>
        </tr>
        <tr>
            <th>GCC ABI</th>
            <th>SysV ABI</th>
        </tr>
    </thead>
    <tbody>
        <!-- Integral -->
        <tr>
            <td rowspan="4" style="text-align: center;">Integral</td>
            <td>char / signed char / unsigned char</td>
            <td>1</td>
            <td>1</td>
            <td>1</td>
        </tr>
        <tr>
            <td>short / signed short / unsigned short</td>
            <td>2</td>
            <td>2</td>
            <td>2</td>
        </tr>
        <tr>
            <td>int / signed int / unsigned int</td>
            <td>4</td>
            <td>2</td>
            <td>4</td>
        </tr>
        <tr>
            <td>long / signed long / unsigned long</td>
            <td>4</td>
            <td>2</td>
            <td>4</td>
        </tr>
        <!-- Pointer -->
        <tr>
            <td style="text-align: center;">Pointer</td>
            <td>any-type * / any function pointer type</td>
            <td>4</td>
            <td>2</td>
            <td>4</td>
        </tr>
        <!-- Floating Point -->
        <tr>
            <td rowspan="3" style="text-align: center;">Floating Point</td>
            <td>float</td>
            <td>4</td>
            <td>2</td>
            <td>4</td>
        </tr>
        <tr>
            <td>double</td>
            <td>8</td>
            <td>2</td>
            <td>8</td>
        </tr>
        <tr>
            <td>long double</td>
            <td>16</td>
            <td>2</td>
            <td>8</td>
        </tr>
    </tbody>
</table>
The biggest difference between GCC and SysV ABIs is, of course, the alignment. While SysV adopts natural alignment, GCC by default aligns larger integer types on 16-bit boundaries.

Note that by using the `-malign-int` flag, which is not enabled by default, GCC aligns `int`, `long`, `float`, `double`, and `long double` on 32-bit boundaries.

Another GCC-specific trick is the `-mshort` flag, which considers `int` to be 16 bits.

### Aggregate Types
Here are the rules to determine the layout and alignment of an aggregate type (for example, `struct` or `union`):
 - The alignment of an aggregate type is the maximum alignment of its element types.
 - Each member will be placed at the lowest offset while respecting to its _member type_ alignment. Internal paddings will be added if needed.
 - The size of an aggregate-type instance should be the multiple of its alignment. Tail paddings will be added if needed.

### Stack Alignment
There is a difference between GCC and SysV ABI on stack alignment: By default, GCC aligns stack data on **16-bit** boundaries, whereas SysV uses **32-bit** alignment.

## Calling Convention
In this section, we're going to talk about the standard [calling convention](https://en.wikipedia.org/wiki/Calling_convention) used by M68k. It is splitted into three sub-sections: Stack frame layout, passing function arguments, and handling return values.

### Stack Frame
The diagram below shows a typical M68k stack frame:
<img style="margin: 30px;" src="/assets/m68k-stack-frame.svg">
Usually, people use special instructions to setup a new stack frame. For instance, `JSR` pushes return address to the stack before jumping to the destination while `RTS` helps you to restore the return address upon returning from a function; `LINK` and `UNLNK` instructions can help you to save and restore the frame pointer.

In addition to special registers like `%fp` and `%sp`, other register usages also follow a certain convention.
<table>
    <thead>
        <th>Register Names</th>
        <th>Usage</th>
    </thead>
    <tbody>
        <tr>
            <td>%d0,%d1 <br> %a0,%a1</td>
            <td>Scratch registers. Caller-save</td>
        </tr>
        <tr>
            <td>%d2 ~ %d7 <br> %a2 ~ %a5</td>
            <td>Local (variable) registers. Callee-save</td>
        </tr>
        <tr>
            <td>%a6 (%fp)</td>
            <td>Frame pointer (if implemented)</td>
        </tr>
        <tr>
            <td>%a7 (%sp)</td>
            <td>Stack pointer</td>
        </tr>
        <tr>
            <td>%pc</td>
            <td>Program counter</td>
        </tr>
        <tr>
            <td>%ccr</td>
            <td>Condition code register</td>
        </tr>
    </tbody>
</table>
For floating point unit, which is available after 68040 / 6881, here is a list of floating point register usages:
<table>
    <thead>
        <th>Register Names</th>
        <th>Usage</th>
    </thead>
    <tbody>
        <tr>
            <td>%fp0, %fp1</td>
            <td>Scratch registers. Caller-save</td>
        </tr>
        <tr>
            <td>%fp2 ~ %fp7</td>
            <td>Local (variable) registers. Callee-save</td>
        </tr>
        <tr>
            <td>%fpcr</td>
            <td>Floating point control register</td>
        </tr>
        <tr>
            <td>%fpsr</td>
            <td>Floating point status register</td>
        </tr>
        <tr>
            <td>%fpiar</td>
            <td>Floating point instruction address register</td>
        </tr>
    </tbody>
</table>

### Function Arguments
As illustrated in the previous diagram, incoming function arguments are passed by stack. Since arguments are always aligned on 32-bit boundaries, they can be accessed at the following memory address (assuming frame pointer is implemented):
```
%fp + 8 + N * 4
```
where `N` is the index of the argument. For instance, the first argument is at `%fp + 8` and the second is at `%fp + 12`.

When passing an aggregate-type object, regardless of their original type alignment, the object will be aligned on 32-bit boundaries.

### Return Values
An integral return value is put in `%d0`, whereas a pointer return value is put in `%a0`.

When it comes to returning an aggregate-type object, the object should be stored in memory and its address will be put in:
  - SysV ABI: `%a0`
  - GCC ABI: `%a1`

Whether it's caller or callee's responsibility to allocate the space remains unspecified.