# Design principles

Each principle has a *because*. A principle without a reason is just a
preference, and preferences don't survive contact with hard trade-offs.

## No ambient authority

Every operation a process performs, it performs because it holds a capability
for it. There is no global namespace it can reach into, no implicit permission
from being "root". **Because** this turns "what can this process do?" from an
open-ended audit into a finite, inspectable list.

## Drivers and policy live in user space

The kernel provides mechanism — address spaces, threads, IPC, capability
operations. Scheduling *policy*, paging *policy*, filesystems, the network
stack, and every device driver run as user-space processes. **Because** a small
kernel is one you can hope to verify, and a driver that can't touch kernel
memory is a driver whose bugs stay local.

## Syscalls are the slow path

Control operations go through traps. High-frequency I/O goes through
shared-memory submission/completion rings, with the trap reserved for waking a
sleeping peer. **Because** at scale the trap cost dominates, and the privilege
boundary doesn't have to.

See [ADR-0003](../adr/0003-ring-based-syscall-interface.md) — this is the
design's biggest bet and the one most likely to be wrong.

## Authority is accountable memory

Kernel objects are created by retyping user-provided untyped memory (the seL4
model). The kernel has no internal heap. **Because** that makes memory fully
accountable, removes in-kernel OOM as a failure mode, and keeps the kernel's
own footprint bounded and inspectable.

## Security is a property, not a layer

Capabilities, DICE-based boot attestation, MTE/PAC/BTI, and SMMU-scoped DMA are
not a "security subsystem" bolted on the side — they cut across every band of
the design. **Because** security added at the edges is security that leaks at
the seams.

## CLI only, forever

No GUI subsystem will be added. **Because** every subsystem we don't ship is one
we don't have to secure or maintain, and the project's value is in the kernel,
not in pixels.
