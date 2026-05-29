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
- **Security surface — and io_uring's record here is genuinely alarming.**
  Shared-memory interfaces between privilege domains have a bad track record.
  io_uring specifically: CVE-2021-3491 (heap overflow → RCE), CVE-2023-1872
  (use-after-free → local privilege escalation), CVE-2023-2598, among others.
  Google reported that **~60% of kernel exploits submitted to its 2022 bug
  bounty targeted io_uring**, and as a result io_uring was *disabled* for apps on
  Android, disabled entirely on ChromeOS, and restricted on Google's production
  servers. We are adopting the interface pattern that one of the largest Linux
  operators decided to switch off. Every field the kernel reads from a
  user-writable ring is an attack surface that must be validated carefully
  (TOCTOU: validate-then-copy, never trust-in-place).
- **Capability checks still cost something.** The ring removes the trap, not the
  authorization check; we must ensure the check is cheap without being unsafe.

## Why this is marked "contested"

This is the decision most likely to be wrong, and we want it challenged. The
honest alternative — a clean trap-based interface — is simpler, smaller, and
easier to verify, at the cost of throughput we haven't measured because nothing
runs yet. If the complexity/security cost outweighs the (unmeasured) performance
benefit, this ADR should be superseded.

There's a sharper version of the tension worth stating plainly: the whole pitch
of this project is a **small, verifiable TCB** ([ADR-0001](0001-pure-capability-model.md)).
A user-writable ring that the kernel reads, with weak-memory ordering and TOCTOU
hazards, is precisely the kind of code that is *hard to verify* and historically
*easy to get wrong*. So this decision may not just be a performance trade-off —
it may be in direct conflict with the project's reason to exist. That's why it's
the loudest open question we have.

**If you read one thing in this repo and disagree with it, make it this.**

## Validation path (before this becomes load-bearing)

This ADR is "accepted" only as the working assumption, not as a settled fact. It
should not be treated as final until:

1. **A prototype exists** of both the ring path and a plain trap-per-syscall
   path, measured for IPC/round-trip throughput and latency on a named target
   SoC — not on x86, where the memory model flatters the design.
2. **A written memory-ordering specification** says which acquire/release
   barriers go where on the ARM weak model, and what they cost. (io_uring has
   itself shipped ordering bugs; we don't get to wave this away.)
3. **A TOCTOU-safe access discipline** for ring fields is specified and, ideally,
   shown to be amenable to the verification approach we choose.

If the prototype shows the throughput win is small, or the verification cost is
large, the right move is to supersede this ADR with the trap-based alternative.
We would rather ship a slower kernel we can trust than a faster one we can't.

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

## Sources

- io_uring(7) manual page (man7.org) — ring buffer design and rationale.
- io_uring CVEs: CVE-2021-3491, CVE-2023-1872 (NVD), CVE-2023-2598.
- "io_uring: Linux Performance Boost or Security Headache?" (Upwind) and the
  2024 HN discussion on io_uring being disabled across Android / ChromeOS /
  Google servers — the ~60% bug-bounty figure.
- seL4 IPC fastpath case study: "Correct, Fast, Maintainable — Choose Any
  Three!" (Trustworthy Systems) for the trap-based baseline this is measured
  against.
