# ADR-0004: No native loadable kernel modules

- **Status:** Accepted
- **Date:** 2026-05-29
- **Deciders:** @AwaleSagar

## Context

Systems need extensibility: new drivers, new filesystems, new policy and
observability hooks. The traditional mechanism is the loadable kernel module —
native code injected into the kernel's address space at runtime. That mechanism
is fundamentally at odds with a small, trustworthy TCB: every loaded module is
unverified privileged code, and the kernel's security properties become
unknowable the moment one is loaded.

## Decision

AscendOS will have **no native loadable kernel modules, ever**. Extensibility is
provided two ways instead:

1. **In user space** — new drivers, filesystems, and services are just new EL0
   processes (this is already how the architecture works).
2. **Through a sandboxed bytecode VM** — for in-kernel-adjacent hooks
   (observability probes, small policy decisions), a verified, restricted
   bytecode (eBPF-class) runs under a verifier with bounded semantics, never
   native privileged code.

## Consequences

**Easier:**

- The TCB stays fixed and analysable; loading an extension can't silently expand
  it.
- Extensions are isolated by construction (process boundary) or bounded by
  construction (verified bytecode).

**Harder:**

- We must build and maintain a bytecode VM and verifier — itself security-
  sensitive code, though far smaller and more constrained than arbitrary native
  modules.
- A few things that are trivial with native modules (deep ad-hoc kernel hooks)
  become deliberately impossible. We accept that.

## Alternatives considered

- **Native loadable modules (Linux-style).** Rejected: incompatible with a
  small, trustworthy TCB; the central thing this project is trying to avoid.
- **Signed-only native modules.** Rejected: signing proves provenance, not
  safety. A signed buggy module is still privileged unverified code in the TCB.
- **No extensibility mechanism at all.** Rejected: observability and policy hooks
  have real value, and forcing everything into separate processes is sometimes
  too coarse. The bounded bytecode VM is the compromise that preserves the TCB.
