# ADR-0002: Device drivers run in user space

- **Status:** Accepted
- **Date:** 2026-05-29
- **Deciders:** @AwaleSagar

## Context

Drivers are, empirically, where a large fraction of kernel bugs and
vulnerabilities live. In a monolithic kernel a driver bug is a kernel bug: it
runs with full privilege and can corrupt anything. We're targeting a small,
isolatable TCB on hardware (ARMv8/ARMv9) that provides an SMMU for DMA isolation
and cheap privilege transitions — exactly the primitives that make user-space
drivers practical, as Fuchsia and the microkernel lineage have shown.

## Decision

Every device driver is an ordinary **EL0 process**, confined to exactly the
MMIO, IRQ, and DMA capabilities granted to it by `devmgr`. Interrupts are
delivered as notifications; MMIO is mapped as frames; all DMA is translated and
bounded by the **SMMU**. Driver crashes are supervised and restarted by `svcd`.

## Consequences

**Easier:**

- A faulty or compromised driver cannot corrupt the kernel or other drivers — it
  holds only its own resources.
- Drivers can be developed, crashed, restarted, and even updated without taking
  the system down.
- The kernel stays small; no driver code in the TCB.

**Harder:**

- Performance: driver I/O crosses a protection boundary. This is precisely why
  the design pairs this decision with the ring-based interface
  ([ADR-0003](0003-ring-based-syscall-interface.md)) and zero-copy shared frames.
- Some latency-critical or tightly-timed hardware may be awkward to drive from
  EL0; we accept needing case-by-case work and possibly a small number of
  kernel-assisted fast paths (to be justified by their own ADRs if ever needed).
- Requires a working SMMU; boards without one weaken the DMA-isolation guarantee.

## Alternatives considered

- **In-kernel drivers (monolithic).** Rejected: directly contradicts the
  small-TCB and isolation goals; reintroduces the largest source of kernel bugs.
- **In-kernel drivers written in a safe language** (safety without isolation).
  Rejected: memory safety helps but a logic bug or a bad DMA descriptor in a
  privileged driver still endangers the whole system. Isolation is a stronger,
  more uniform guarantee.
- **Hybrid: critical drivers in-kernel, the rest in user space.** Rejected for
  now: the boundary becomes a judgment call that erodes over time. We'd rather
  start pure and add narrow, individually-justified exceptions only if measured
  need demands it.
