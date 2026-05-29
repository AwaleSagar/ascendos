# Scheduling

The scheduler is mechanism in the kernel; policy knobs are exposed to user space
through scheduling-context capabilities.

- **EEVDF-style** virtual-deadline fairness rather than classic CFS heuristics.
- **Capacity-aware** placement for ARM DynamIQ big.LITTLE — E-cores vs P-cores.
- **Scheduling contexts as capabilities** — a thread runs only against a budget
  it holds, which is what makes hard real-time and safe over-commit expressible.
- Per-CPU runqueues with same-cluster work-stealing; tickless idle.

## A real tension: capabilities vs. bounded real-time

We claim *both* pure capabilities ([ADR-0001](../adr/0001-pure-capability-model.md))
*and* hard real-time via scheduling-context capabilities. Those pull against
each other, and we shouldn't pretend otherwise. Recursive capability
**revocation** is a long-running operation (seL4 deliberately breaks it into
preemptible sub-operations), and dynamic capability creation makes it hard to
*bound* sub-CDT size and revocation cost — which is exactly what a real-time
latency guarantee needs. Recent work (e.g. RTSS 2025 on partitioning kernels
with capability-controlled isolation) calls this out directly as a weakness of
the seL4-style dynamic model. For AscendOS this is an open design problem, not a
solved one: bounding worst-case revocation latency under the MCS model needs a
concrete answer before the real-time claim is credible.

## Open questions

- How do scheduling-context budgets compose across a call chain through several
  servers? Budget donation across IPC needs a concrete design.
- How do we bound worst-case revocation latency so it doesn't blow the real-time
  guarantee? (See the tension above.)

## On precision

"EEVDF-style" is deliberately loose right now and that's a gap. The eventual ADR
(issue #3) needs to say whether we implement the original Stoica/Abdel-Wahab
algorithm, the Linux 6.6 adaptation, or a custom variant — and pin down how
per-core capacity is measured and how tasks are classified for big.LITTLE
placement.

Full detail: blueprint §4.1. The choice of EEVDF over CFS-style heuristics is
informed by recent Linux scheduler work but not yet written up as its own ADR —
that's a gap (issue #3).
