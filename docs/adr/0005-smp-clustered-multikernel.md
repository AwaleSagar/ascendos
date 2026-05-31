# ADR-0005: SMP as a clustered multikernel, not a shared concurrent kernel

- **Status:** Accepted
- **Date:** 2026-05-31
- **Deciders:** @AwaleSagar

## Context

The blueprint wants two things that are in tension on multicore hardware: a
**small, verifiable kernel** ([ADR-0001](0001-pure-capability-model.md)) and
**SMP scalability**. The original framing (§16) treated SMP as a smooth step —
"single core → SMP → NUMA-aware → optional multikernel" — with per-CPU runqueues
and work-stealing, implying a single shared kernel mutated concurrently by all
cores.

That implication is the problem. A single shared kernel touched concurrently by
many cores needs fine-grained locking, and fine-grained concurrent kernel code is
— today — beyond tractable formal verification.

## Evidence

- **seL4 FAQ:** "The SMP kernel uses a big-lock approach … verification of the
  multicore kernel is in progress for static multikernel configurations." Even
  the most mature verified kernel does **not** have a verified fine-grained SMP
  kernel.
- **seL4 Summit 2022/2023 (McLeod):** explicit design constraints — "No
  concurrency in the kernel. No changes to verified C code." The path to verified
  multicore they pursue is replication, not shared mutation.
- **"The clustered multikernel" (UNSW thesis):** reduces concurrent data access
  "to a minimum while offering a configurable trade-off between scalability and"
  verifiability.
- **Barrelfish multikernel (SOSP 2009):** the canonical design — treat the
  machine as a network of cores, replicate OS state per core, communicate by
  explicit message passing.

## Decision

The **primary SMP model is a clustered multikernel**: one single-threaded kernel
instance per core (or per cluster), each owning a replica of kernel state, with
cross-core coordination by **explicit message passing**, not shared-memory
mutation of a single kernel. Because each instance is single-threaded, it is
big-lock-free *by construction* and stays within reach of verification.

A single shared kernel with fine-grained internal locking is **rejected**.

## Consequences

**Easier:**

- Each kernel instance stays single-threaded → no in-kernel concurrency → the
  verification story of [ADR-0001](0001-pure-capability-model.md) survives onto
  multicore.
- Maps cleanly onto big.LITTLE clusters and NUMA.

**Harder:**

- Cross-core operations become explicit messages — higher latency than a shared
  data structure, and a consistency protocol to design.
- State replication costs memory and cache footprint; placement matters (see
  [F9 / scheduling](../architecture/scheduling.md)).
- User-space services that want to span cores must be written for message-passing,
  not shared memory (SMP-like user apps, per seL4 Summit 2023).

## Alternatives considered

- **Single shared kernel, big lock.** Verifiable but caps multicore throughput
  (the seL4 SMP status quo). Rejected as the *target* — acceptable only as an
  early bring-up crutch.
- **Single shared kernel, fine-grained locking.** Scales but is currently
  unverifiable; directly contradicts the project thesis. Rejected.
- **Clustered multikernel (chosen).** Best preserves verifiability while scaling;
  the only model with credible verified-multicore precedent.
