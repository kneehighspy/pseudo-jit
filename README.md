# Pseudo JIT

## A Brief bit of Background

Threading is a term borrowed from the FORTH programming language. Without introducing too much terminology here, in traditional FORTH a thread is simply a list of addresses either pointing to machine code or another list of addresses. There are four common threading models including direct, indirect, subroutine and token threading. If you're interested in the deeper inner workings of FORTH, I would strongly recommend Brad Rodriguez's Moving Forth series [here](https://www.bradrodriguez.com/papers/moving1.htm).

Interpreters are a form of token threading where each opcode is read from instruction memory and the interpreter then branches to the subroutine for that opcode. It is the simplest form of emulation but is seldom the fastest; the overhead of fetching, decoding and branching to the opcode can be quite high and often takes more processing time than the instruction itself.

Recompilers omit the threading overhead by translating blocks of code into native machine instructions and operate on the (mostly correct) assumption that code is executed more than once. Each chunk is recompiled at the entry point up-to the first branch where the new code is expected to return a new entry point to continue the process. Previously recompiled sections are then retrieved from a cache to avoid the overhead of translation.

However, they are not perfect. 

First of all, the time to recompile code can be quite expensive and is much slower than interpretered code. This overhead leads to micro-stutters (jitter, pun not intended) in the execution of code and a worsening of performance for rarely executed code. Recompilers can perform so poorly that most also implement stanadard interpreters for these cases and only use recompilation when obvious loops or frequently used subroutines are known.

Secondly, the code which is produced may be an order of magnitude larger than the original, resulting in a proportional demand on memory. While the cost of memory is relatively cheap, this can impact cache locality and negatively impact performance. This is exacerbated when the emulator recompiles the same region of memory several times over due to different entry points.

And finally, recompilers are quite complex, having to address incongruities of different processor architectures while attempting to emit the most compatible and ideally, fastest code possible. Applying optimizations and attempts to mitigate the first two issues only complicate the process more.

## Threaded JIT

I started this with a brief introduction with the classic FORTH threading models. Token threading is the natural model for an interpreter as the opcodes of the guest processor are equivalent to tokens. In most cases, these tokens are only partially resolved by the token threader and some limited opcode decoding is still performed within the subroutine. Direct, indirect and subroutine threading have generally been avoided entirely.

Peter McGavin is the first I'm aware of to implement a threaded JIT in his Z80 emulator. It works by creating a 64K entry jump table where each entry corresponds to the opcode taken from instruction memory. Upon initialization, this jump table points to the opcode lookup routine itself, which performs in-place substitution to the actual opcode handler before jumping to it directly. When code reaches this point a second time, the opcode handler is executed immediately without reexamining the instruction memory.

Richard Carlsson noted in his adaptation of this approach in his Z80 emulator, that the resulting speed up over a traditional emulator is about two-fold.

As with FORTH, in this model there is no longer a traditional interpreter; at best, the opcode look up routine serves in its place. And while that function may have some overhead, the execution of individual instructions still occurs sequentially, eliminating the problem of jitter -- programs are now easily deterministic. Even if a threaded JIT is not as fast as a normal JIT, it is **quicker**.

Secondly, because the opcode is only replaced with a single subroutine call or address, there's a fixed "expansion ratio" that is not only much smaller than code generated by a recompiler but having a fixed ratio means that branches and jumps are now trivial to compute and the emulator can freely jump into any address in memory whether it has been precompiled or not.

Finally, compared to a traditional recompiler, a threaded JIT is trivial to implement. More on that later.

Despite its performance over interpreters and even recompilers in some cases, threaded JIT did not become popular and was largely forgotten. The overhead of a subroutine call and return on each instruction was too great and the performance gain was not as impressive as normal JIT -- especially when up against benchmarks which unfairly exaggerate the benefit of JIT. This simple model also scales pooly in 32-bit systems trying to emulate other 32-bit systems as there simply isn't enough memory for the jump table.

## Enter PJIT, the Pseudo-JIT

As discussed, Threaded JIT faces two major problems: poor scaling and subroutine overhead.

Subroutine overhead has surprisingly already been solved for us by CPU manufacturers. In the race for more performance and to mitigate the cost of a pipeline stall, all modern processors include some form of branch prediction and return stack caching. The overhead of a subroutine call is now virtually zero in most cases.

YAY!

We can mitigate these even further with some simple inlining -- for example, any instruction that would generate an exception will simply jump straight to the exception routine avoiding a costly branch-to-a-branch, and the NOP instruction can be replaced with a NOP to avoid any branching at all. Any more complicated substitutions are avoided to keep the opcode translator simple.

The other issue is the scaling of the cache; a 32-bit system would need a 4 billion entry jump table and if each address were a single address and jump instruction, the resulting table alone would need many times that, which even modern systems would struggle to manage. It's also a nonsensical approach, since it's very unlikely that all of that memory would be used for instructions -- far more of it would be used for data.

Instead, a more adaptive set-associative cache was added. This breaks the available cache space into pages which correspond to some memory in the host memory -- aligned, this still leads to a trivial bitshift to translate between the two. There are no linked lists and no dynamic allocation, the cache and all structures required for it can be statically allocated. In cases where a cache miss occurs, the performance penalty for recompilation is trivial compared to a traditional JIT.

This is no longer a straight Threaded JIT, hence the name Pseudo JIT (PJIT).

# Summary of PJIT

PJIT is a real-time version of JIT that avoids the traditional issues of dynamic memory allocation and non-determinism. It is designed to run as a bare metal process on ARM microprocessors operating in big endian mode to provide transparent access to a traditional 68K bus. It is useable with ARM SoC that integrate only a modest (by today's standards) amount of memory. 

PJIT does not attempt to be a full system emulator, however, where required, the low level bus arbitration of the 68000 itself is emulated.

PJIT does not provide any OS type restrictions on the guest system nor does it grant any obvious benefit such as being able to call drivers or libraries of a hypothetical host OS.

## PJIT Organization

PJIT consists of two major tables: the opcode thread cache (simply 'the cache') and the opcode literal table (simply 'the table').

### The Cache

The cache is constructed as an n-way set associative cache where the n-weight is a compile-time option to be determined during performance testing later. It will use roughly 1/8 to 1/2 all of available system memory, with the final actual size to be determined later -- a large cache might be underutilized with the typical size of executables to be found on the Amiga platform and a cache that is too small may incur too many cache misses that reduce performance.

Each cache page is seaprated by a block of return statements to the outer interpreter to handle the variable opcode lengths of the 68000.

### The Table

The table consists of every opcode permutation as a single, algorithmically generated function. Each function can then assume to know before-hand the sum of all it's registers, addressing modes and operations, save for when the so-called extended addressing modes are used or when handling co-processor operations.

## PJIT Processes

PJIT consists of two major processes: the outer interpreter and the inner interpreter.

### The Inner Interpreter

The inner interpreter is subroutine threaded where each opcode is implemented in the cache as a soubroutine call and each operation is terminated with a return statement. Because of branch-prediction and the return-link stack; the overhead of subroutine threading should be negligible and should exceed any other threading model save for traditional JIT.

### Operands

The original 68000 stream is a combination of opcodes and operands and when returning to the subroutine thread, it's important to not execute operands as instructions. Since modifying the link register would cause a pipeline stall, operands shall instead be replaced with up to three NOPs (for 1-3 operands) or one NOP followed by a branch (for 4 or more operands) with NOPs padding the gap.

### The Outer Interpreter

The outer interpreter[^1] will find the entry point into the cache and if necessary, invalidate the page when a cache miss occurs. In this case, the cache is preloaded with the address of the opcode lookup function. This function will use the current real program counter to determine what the 68000 program counter ought to be, looks up the opcode in the table and substitutes the cache entry with a call to the opcode itself. It then calls the opcode immediately before returning.

### Opcode Handlers

All normal (straight-line) opcodes within the table simply end in a return statement; simply 'ld pc, lr'. The link register also performs double duty as a way of quickly determining what the original 68K program counter should be.

All non-conditional branching opcodes within the table simply jump to the the outer interpreter where the entry point is rechecked against the cache tag before continuing.

All conditional branches within the table will either return (branch not-taken) or jump to the outer interpreter (branch taken) as required.

## (Possible) PJIT Optimizations

### Branch and Return to NOP

The ARM processor does not handle a branch-to-a-branch condition; it is always preferable to branch to an instruction. With that in mind, it may make sense to increase the size of the cache by two to interleave NOP and BL commands with the return inevitably connecting with the subsequent NOP and not another branch. 

### Flag Elimination

On the 68000, nearly every instruction can set various flags in the condition register and these have to be captured in the emulator. However, if it is clear that the very next instruction resets the same flags that this instruction sets, then a version of the opcode which does not set any flags may be used. This often omits one or two instructions from each and every opcode

### Short Branches

If a branch or jump is guaranteed to be in the same cache page then its possible to do a direct branch without returning to the outer interpreter. This will incur some branch misprediction performance, but should still be faster.

### Selective Inlining

Some subroutines may be no bigger than the subroutine call to them and this is certainly the case with NOP. With NOP padding, this may be even more true. A simple table of each opcode's size can be crafted which is then used to determine if inlining is possible and how much NOP padding is needed.

# Notes

[^1]: In classic Forth, the outer interpreter is always constructed of statements of the inner interpreter which means that the true state of the machine never leaves of inner interpreter context; with a machine implementation of the inner interpreter, this model doesn't make sense, so the outer interpreter shall be regular ARM code as compiled by the C compiler.
