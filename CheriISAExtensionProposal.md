# CHERI ISA Extensions for Temporal Stack Safety Proposal

*This document contains an explanation of my proposed changes to the CHERI-RISC-V ISA to improve temporal safety for stack-allocated addresses. I typed up [a proposal document](https://github.com/bencole12345/CheriISAExtensionProposal/blob/main/Ben%20Cole%20-%20Temporal%20Stack%20Safety%20Proposal.pdf) a few weeks ago, but it's a bit longer than this and I have since deviated a bit from it.*

## Links

- [Sail model of these changes](https://github.com/CTSRD-CHERI/sail-cheri-riscv/compare/master...bencole12345:ben-stack-temporal-safety)

  I haven't yet tested this as extensively as I would have liked, but the general idea should hopefully be clear.

- [Work-in-progress Clang implementation](https://github.com/CTSRD-CHERI/llvm-project/compare/master...bencole12345:temporal-stack-safety)

  This includes a new pass that scans through and inserts `ccsc` instructions in front of all pointer-type stores, as well as routines to insert stack alignment code at the beginning of functions.

- [HIGHLY work-in-progress QEMU implementation](https://github.com/CTSRD-CHERI/qemu/compare/qemu-cheri...bencole12345:ben-stack-temporal-safety)

  There's currently a bug with my QEMU branch in which the `is_heap_capability()` function's output disagrees with the output from GDB's `info registers` command. Will update this page once it's fixed!

## Aim and motivation

My aim is to add architectural support for stack temporal safety to CHERI. Existing papers have already described the possibility of [encoding lifetimes into capabilities](https://ieeexplore.ieee.org/document/8823727) or [using the capability addresses themselves as a measure of lifetime](https://popl21.sigplan.org/details/prisc-2021-papers/2/Toward-Complete-Stack-Safety-for-Capability-Machines), and forbidding stores of capabilities “in the wrong direction” so that it becomes impossible for a capability pointing to a deallocated stack address to even exist. If a caller can't store a capability to a callee's stack frame, then it can never accidentally dereference it out of lifetime.

These schemes provably provide full stack temporal safety, but come with a few disadvantages:

1. They would disallow some legal (albeit awkward) C and C++ code.

2. Encoding lifetimes into capabilities (the first paper) requires quite a lot of bits to cover the full range of possible call depths -- the authors suggested 16.

3. Disallowing storing capabilities to shorter-lived *addresses* (the second paper) would break even more valid code, since it would prevent circular references within a stack frame.

## My solution

My proposed solution improves on these in two ways:

1. It offers full compatibility with all legal C and C++ code. It does this by using a “fast path” with architecturally-visible lifetimes as much as possible, but falling back to a slower scheme when the architectural lifetimes cannot prove a store instruction to be safe. Even if this is horribly slow, the hope is that it would be rare, and it would still be an improvement over the previously-proposed schemes that would have rejected valid code.

2. In addition, I have devised a lifetime encoding scheme that uses far fewer per-capability bits than explicitly encoding capabilities with their lifetime value (stack depth), at the cost of less efficient stack-memory usage caused by the stronger alignment requirements.

I propose to reserve three bits per capability to encode stack-frame sizes. There would be seven power-of-two values and an eighth “I'm not a stack capability” state. I'm yet to choose final values, but 64 bytes up to 4 KB is what I've been using so far. Stack frame sizes unfortunately need to be implicitly rounded up to the next power-of-two -- this is where the memory wastage happens. At the beginning of each function, the compiler would emit code to align the stack pointer to a number of bits determined by the function's frame size, saving the original stack pointer so that it can be restored afterwards.

The result is that, given any capability, one can determine unambiguously whether it points to a stack address, and if so, the address range of the stack frame in question. Because stack frames are now aligned, and because stack capabilities now specify the size of the frame they are pointing to, you can now determine unambiguously for two stack capabilities which one is "higher" in the call frame without having to record their stack depths explicitly. You can also tell when two capabilities point to the same frame, meaning that you don't need to disallow circular references within one.

To take advantage of this, a new instruction called `ccsc` (Check Cap Store Cap) should be issued by the compiler before any `csc` (Cap Store Cap) instruction. (Later I'll mention possible optimisations to elide this.) This instruction:

- Reads the stack frame size bits of the source and destination registers;

- Generates an appropriate AND mask for each;

- Masks the destination and source addresses with their AND masks;

- Raises a new exception, `StackLifetimeViolation`, if the destination's implied lifetime is shorter than the source's lifetime.

- (Plus a bit more logic for dealing with non-stack capabilities. Capabilities to the stack cannot be stored on the heap, and capabilities from the stack to the heap should always be allowed.)

## What to do when the exception fires

If you let a `StackLifetimeViolation` exception terminate the process, then, ignoring the case in which a stack frame is too large, you would have the same level of security and compatibility as the capability lifetimes scheme (the first link). However, we can do better!

The next step is to implement a `StackLifetimeViolation` trap handler routine that causes a Cornucopia revocation sweep to be invoked at the end of the function. It should set a bit for that stack frame to indicate that an exception has been raised, which is read by code in the epilogue. The revocation sweep needs to be synchronous because the stack addresses are likely to be reused immediately by the next function call.

Again, this will inevitably be slow, but it's better than rejecting legal code, and with a good escape-analysis pass, programmers would be able to see at compile-time that this might be the case and would have the option to move possibly-escaping stack variables to the heap. Revocation sweeps would also be required for stack frames that are too big to describe using these three bits, and I'm yet to determine how to handle dynamic stack allocations. It might be interesting to explore the safety of letting programmer annotations assert that a variable is non-escaping.

## Summary of ISA changes

- Add a new instruction, `ccsc`, that takes the same operand format as a `csc` instruction.

- Add a new exception, `StackLifetimeViolation`, to be thrown by the `ccsc` instruction if the address lifetimes are found to be oriented incorrectly.

- Add a second new instruction, `CSetStackFrameSizeImm`, that sets the frame size bits for a capability. The compiler would emit this in the prologue of a function.

## Possible optimisations

- Functions without any escaping local variables need not be aligned nor include any `ccsc` instructions. Hence, with an accurate escape-analysis pass, there would be no overhead (at least, architecturally) for “well-behaved” functions. (There might still be microarchitectural overheads if the additional `ccsc` instructions and larger function prologues and epilogues damage I-cache performance.)

- Smarter data-flow analysis would enable the compiler to narrow down which stores within a function may actually include the escaping value, reducing the number of `ccsc` instructions that need to be issued.
