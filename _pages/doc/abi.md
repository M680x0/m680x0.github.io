---
title: "M68k Application Binary Interface (ABI)"
toc: true
---
## Variants of ABIs
In this document, we're covering two variants of M68k ABIs:
  1. **GNU / GCC ABI**: The default ABI used by GCC. It is backward-compatible with many ABIs used by legacy compilers, such as Sun [PCC](https://en.wikipedia.org/wiki/Portable_C_Compiler). Yet it lacks formal documentations. This is currently the only ABI implemented by M68k LLVM.
  2. **SystemV ABI**: The default ABI used by UNIX systems. There is a nice [book](https://github.com/M680x0/Literature/raw/master/sysv-m68k-abi.pdf) documenting related details. Most of the contents in this page are also adapted from it.

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
            <td>any-type * / any-type (*)()</td>
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
_TBA..._

### Stack Frame

### Function Arguments

### Return Values