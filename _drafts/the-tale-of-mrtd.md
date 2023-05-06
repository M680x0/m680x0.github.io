---
title:  "The tale of `-mrtd` in GCC and Clang"
layout: single
classes: wide
author: mshockwave
---

Recently I've been working on an [issue](https://github.com/llvm/llvm-project/issues/60554) about supporting the [`RTD`](/ref/integer-instructions.html#pf10e) instruction in m68k LLVM backend (well, it turned out to be not directly related to the instruction itself but we will talk about that later).
The gist of that issue goes like this: the backend bailed out when it tried to lower a certain kind of return statement to `RTD`, if we're targeting 68020 or later.

`RTD` is a variant of return instruction that subtracts the number of bytes, indicated by its immediate-value operand from the stack pointer (which effectively pops the stack) before returning.
It can be used to implement a special kind of calling convention in which the callee has to clean out the space allocated for arguments passed from the stack.
In m68k GCC, this calling convention is not enabled by default unless the `-mrtd` flag is present.
Since this project is aiming to be compatible with its GCC counterpart (and the fact that this calling convention was not commonly used even in the good ol' days), we want to implement the same behavior.

Cool, I guess we need to add `-mrtd` to Clang. Because given "rtd" being such an odd name and sounds really 68k-specific, there is no way it's already there...

> "So, I have been digging into this now and what I found is that `-mno-rtd` is actually already handled by Clang:
> 
> `def mrtd: Flag<["-"], "mrtd">, Group<m_Group>;`
> 
> and
> 
> `def mno_rtd: Flag<["-"], "mno-rtd">, Group<m_Group>;`
> 
> Also documented here: https://clang.llvm.org/docs/ClangCommandLineReference.html"

_-- Quoted (and slightly edited) from [one of the comments](https://github.com/llvm/llvm-project/issues/60554#issuecomment-1519005515) by Adrian._

So `-mrtd` is _already_ there? That's a little weird...

It seems like X86 is the only user of that flag. Specifically, only the 32-bit i386 which uses it to enable the `stdcall` calling convention, a CC that also requires callee functions to pop out incoming arguments on the stack.
`stdcall` is primarily adopted by Win32 API.

Aside from `stdcall`'s similarity with our special CC mentioned earlier, it's not quite obvious why the flag is named after something unrelated to i386.

Rest of this post is served to answer a simple question: *why do i386 Clang and GCC use the name "(m)rtd?"*.
It's important to note that this article is **NOT** meant to criticize / compare any naming choice or convention made in the past between different compiler backends and implementations, but merely a historical study.

# Why is it called "(m)rtd"?

The first assumption I came out was that maybe there _is_ an instruction called "rtd" in i386, specifically the old 80386 processor.
Given how iron-fisting Intel is on maintaining backward compatibility, it's nearly impossible that any instruction has been removed from the ISA since 80386. Therefore, looking up a relatively modern 32-bit x86 ISA manual should suffice.

Unfortunately, a simple search will tell you that i386 only has `RET` and `RETI` (return from interrupt).
`RET` _does_ have a variant that takes an immediate-value operand, acting just like `RTD` in 68k we mentioned earlier. But, well, it's still called "ret" rather than "rtd".

Now, maybe there are some clues in the patch that introduced this flag to Clang or even GCC -- time to dig into the past.

# Dragon archaeology

Let's start from the Clang/LLVM side. The `-mrtd` flag was added to Clang by [65b88cd](https://github.com/llvm/llvm-project/commit/65b88cdb3bd34e5000b30533fa1599c959029719) in 2011.
Unfortunately, there wasn't any commit message or code comment attached to shed some lights on the choice of flag name.
But luckily Clang, as a compiler driver, is supposed to be [compatible with GCC](https://clang.llvm.org/docs/DriverInternals.html#gcc-compatibility). So one can safely assume that this flag is originated from GCC.

Digging into GCC's source code, at hindsight [6ac4959](https://gcc.gnu.org/git/?p=gcc.git;a=commit;h=6ac49599123d5d649107b8d7fd0674a8fb1afdbe) added `-mrtd` to i386 GCC in 1992, residing in file `config/i386/i386.opt`.
But if we look closer, that patch was merely transferring flag declarations to the newer generator-based approach using *.opt files.
The [original definition](https://gcc.gnu.org/git?p=gcc.git;a=blob;f=gcc/config/i386/i386.h;h=5854944c8ec2b7b054a6c9f3aacb35778c46e67c;hb=0e5d569cd56e49dd5be9a67d553f0c007ff5436c#l357) of `-mrtd` flag in `config/i386/i386.h` can actually be traced all the way back to the _initial_ version of i386 backend!
Specifically, [c98f874](https://gcc.gnu.org/git/?p=gcc.git;a=commit;h=c98f874233428d7e6ba83def7842fd703ac0ddf1) authored on **Feb 9th 1992**.

# ~~Dragon~~Gnu archaeology

Here was my assumption:
> The name "-mrtd" in i386 GCC was reused or copied from its m68k counterpart.  

So I looked into the first commit that introduced `-mrtd` to m68k GCC, which was [3d339ad](https://gcc.gnu.org/git/?p=gcc.git;a=commit;h=3d339ad2b6fa7849631f4cf485efb71638687981).
But the timestamp showed that it was authored on **Feb 18th 1992**, 9 days _after_ the first file in the initial version of i386 GCC!

## "LookMaNoVCS_FINALv9.4FINALFINALrev87.psd"

It turns out GCC not only has used several different VCS (Version Control System) [in the past](https://mpoquet.github.io/blog/2020-08-vcs-adoption-in-floss/index.html), it was not even managed with a VCS at the very beginning.
According to [GCC's History](https://gcc.gnu.org/wiki/History), the first (beta) [release](https://groups.google.com/g/mod.compilers/c/ynAVuwR7dPw/m/-IirjtgwPxsJ) was put...on a FTP server located in MIT.

So the Feb 9th 1992 date we just mentioned was merely the time i386 backend was checked into GCC's VCS. Same for the Feb 18th 1992 date of its m68k counterpart.
It's highly likely that the code for i386 and m68k backend was already there before any VCS adoption.
The best way to answer this is to grab GCC's pre-VCS era source code. Unfortunately, while the [FTP server](http://prep.ai.mit.edu/gnu/) that originally hosted GCC is still there, I no longer can find that particular copy of source code.
We can only make some educated guesses now.

There are three pieces of clues I found useful here:
  1. `-mrtd` was also used for one of the obselete (and ancient) architectures called Gmicro. A comment about `-mrtd` in Gmicro backend [said](https://gcc.gnu.org/git?p=gcc.git;a=blob;f=gcc/config/gmicro/gmicro.h;h=3d50048a13a0dded561039162d15e0acd4ec6e91;hb=44f0c3edadbe3baa3ed045a4a9917719dd65029b#l461): _"...On the m68k this is an RTD option, so I use the same name for the Gmicro. The option name may be changed in the future."_
  2. The initial version of `config/i386/i386.h` and `config/m68k/m68k.h` shared a nearly identical line of the comments related to `-mrtd` handlings ([i386 line](https://gcc.gnu.org/git/?p=gcc.git;a=blob;f=gcc/config/i386/i386.h;h=c0cd287b3d2f0c5d66baf1b8ed168239639771e9;hb=c98f874233428d7e6ba83def7842fd703ac0ddf1#l517) v.s. [m68k line](https://gcc.gnu.org/git/?p=gcc.git;a=blob;f=gcc/config/m68k/m68k.h;h=2fea1a12357cfc0ca2cce6fc3351cebc3c9429a8;hb=3d339ad2b6fa7849631f4cf485efb71638687981#l739)). The only difference between them is the supported processor name (i.e. "80386" v.s. "68010"). 
  3. From the [announcement](https://groups.google.com/g/mod.compilers/c/ynAVuwR7dPw/m/-IirjtgwPxsJ) of the first GCC beta release made by RMS (circa March 1987), it's high likely that m68k and VAX were the only two supported targets.

Item (1) suggests that reusing a flag name, despite having little to do with the respective instruction name (in Gmicro the corresponding instruction is called `EXITN` not `RTD`) was a thing, and might even be a common practice;
item (2) is likely to be a trail of boilerplate copy-n-paste on not just the comment but also the code, as well as the flag.
Finally, item (3) further affirms that if both (1) and (2) hold, it's likely that `-mrtd` was reused or copied from m68k to i386 GCC rather than the other way around.

# Conclusion
Though there isn't any direct evidence showing that `-mrtd` was borrowed from m68k GCC to i386 GCC (and eventually rippled to Clang) it's very likely the case, supported by the..._artifacts_ I presented.

But in any case, if patch [D149864](https://reviews.llvm.org/D149864) and [D149867](https://reviews.llvm.org/D149867) are accepted in the future, m68k Clang/LLVM will finally recognize `-mrtd` --- nearly **40 years** after its debute in GCC.

A small victory for the m68k LLVM community nonetheless!