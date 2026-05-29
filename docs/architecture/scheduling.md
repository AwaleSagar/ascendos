# Scheduling

The scheduler is mechanism in the kernel; policy knobs are exposed to user space
through scheduling-context capabilities.

- **EEVDF-style** virtual-deadline fairness rather than classic CFS heuristics.
- **Capacity-aware** placement for ARM DynamIQ big.LITTLE — E-cores vs P-cores.
- **Scheduling contexts as capabilities** — a thread runs only against a budget
  it holds, which is what makes hard real-time and safe over-commit expressible.
- Per-CPU runqueues with same-cluster work-stealing; tickless idle.

## Open questions

- How do scheduling-context budgets compose across a call chain through several
  servers? Budget donation across IPC needs a concrete design.

Full detail: blueprint §4.1. The choice of EEVDF over CFS-style heuristics is
informed by recent Linux scheduler work but not yet written up as its own ADR —
that's a gap.
