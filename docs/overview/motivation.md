# Motivation

## The gap

General-purpose kernels have grown enormous. That size buys compatibility and
performance, but it costs something specific: you can no longer hold the whole
trusted computing base in your head, and you certainly can't formally reason
about it. The monolithic model puts drivers, filesystems, and network stacks in
the same privilege domain as the scheduler, so a bug anywhere is a bug
everywhere.

The microkernel lineage answered this decades ago — keep the kernel tiny, push
everything else into user space — and seL4 went further by *proving* a kernel
correct. The cost was historically performance and ergonomics. But two things
changed:

1. **Hardware caught up.** ARMv8/ARMv9 gives us cheap privilege transitions, an
   SMMU for DMA isolation, and memory tagging — the primitives a microkernel
   wants.
2. **The I/O model changed.** `io_uring` showed that the syscall trap, not the
   privilege boundary, was the real bottleneck. Shared-memory completion rings
   make a small-kernel design fast.

## Why ARM64-first, CLI-only

Designing for one architecture family removes a pile of abstraction that exists
only to paper over differences we don't have. ARM64 is where the interesting
isolation hardware lives, and it spans from servers to the smallest boards.

Dropping the GUI is not minimalism for its own sake. A compositor, a font stack,
and a framebuffer pipeline are large attack surfaces and large maintenance
burdens. A CLI-only system is one we can actually finish and secure.

## What success looks like

A system small enough to be a teaching artifact and structured well enough to be
a serious bring-up target later: a kernel whose TCB you can read in an afternoon,
drivers that can crash without taking the system down, and a security model where
"what can this process do?" has a precise answer.

We are not claiming to have built this. We are claiming the design is worth
getting right first. See [Non-goals](non-goals.md) for the boundaries.
