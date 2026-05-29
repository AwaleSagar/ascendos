# ADR-0003: Ring-based syscall interface

- **Status:** Accepted (contested — this is the design's biggest bet)
- **Date:** 2026-05-29
- **Deciders:** @AwaleSagar

## Context

A microkernel with user-space drivers and services moves a lot of work across
protection boundaries. The classic concern with microkernels is IPC/syscall
overhead. But `io_uring` demonstrated on Linux that the dominant cost in modern
I/O is often the **syscall trap itself**, not the privilege boundary, and that
batching requests through shared-memory submission/completion rings can largely
amortise or eliminate it.

We have a choice about the *primary* kernel interface: a conventional
trap-per-operation model, or a ring-based model where the trap is reserved for
waking a sleeping peer.

## Decision

The primary high-frequency interface is a **shared-memory submission/completion
ring** per thread (io_uring-style). Requests are enqueued without trapping; a
trap ("doorbell") is issued only to wake a peer that is idle. A conventional
`SVC` trap path remains for control operations (capability manipulation, setup,
rare calls). Bulk data moves through shared frames, so the kernel sits on the
control path but not the data path.

## Consequences

**Easier / better:**

- High-frequency I/O avoids per-operation trap cost, which is what makes the
  user-space driver/service architecture ([ADR-0002](0002-userspace-drivers.md))
  performant rather than merely safe.
- Natural batching and pipelining.

**Harder / risks:**

- **Complexity in the kernel's hot path.** Ring management, memory ordering on
  ARM's weak model, and wakeup/doorbell logic are subtle and easy to get wrong.
  This adds to the TCB we're trying to keep small — a real tension with
  [ADR-0001](0001-pure-capability-model.md)'s verifiability goal.
- **Security surface.** Shared-memory interfaces between privilege domains have
  had real vulnerabilities (io_uring itself has a notable CVE history). Every
  field the kernel reads from a user-writable ring is an attack surface that must
  be validated carefully (TOCTOU).
- **Capability checks still cost something.** The ring removes the trap, not the
  authorization check; we must ensure the check is cheap without being unsafe.

## Why this is marked "contested"

This is the decision most likely to be wrong, and we want it challenged. The
honest alternative — a clean trap-based interface — is simpler, smaller, and
easier to verify, at the cost of throughput we haven't measured because nothing
runs yet. If the complexity/security cost outweighs the (unmeasured) performance
benefit, this ADR should be superseded.

**If you read one thing in this repo and disagree with it, make it this.**

## Alternatives considered

- **Pure trap-per-syscall (classic seL4-style).** Strongly considered. Smaller,
  simpler, more verifiable. Rejected *as the primary path* on the bet that
  throughput matters for the I/O-heavy user-space architecture — but this is a
  bet, not a measurement, and it's reversible.
- **Rings only, no trap path at all.** Rejected: control operations and the
  cold-start/idle case still need a wakeup primitive; a trap path is the simplest
  way to provide it.
- **Asynchronous message queues without shared-memory rings.** Rejected: doesn't
  eliminate the per-message trap cost that motivated this decision.
