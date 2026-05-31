# Scheduling

The scheduler is mechanism in the kernel; policy knobs are exposed to user space
through scheduling-context capabilities.

- **EEVDF-style** virtual-deadline fairness rather than classic CFS heuristics.
- **Capacity-aware** placement for ARM DynamIQ big.LITTLE — E-cores vs P-cores.
- **Scheduling contexts as capabilities** — a thread runs only against a budget
  it holds, which is what makes hard real-time and safe over-commit expressible.
- Per-CPU runqueues with same-cluster work-stealing; tickless idle.

## SMP model: clustered multikernel, not a shared concurrent kernel

The per-CPU runqueue language above must not be read as "one shared kernel that
all cores mutate concurrently." That model needs fine-grained in-kernel locking,
which is — today — beyond tractable verification. The evidence is direct: even
seL4's SMP kernel uses a **big lock**, and verified multicore is still in
progress (seL4 FAQ). The credible path to verified multicore is **replication**,
not shared mutation.

AscendOS therefore adopts a **clustered multikernel** as its primary SMP model
([ADR-0005](../adr/0005-smp-clustered-multikernel.md)): one single-threaded
kernel instance per core/cluster, each owning a replica of kernel state, with
cross-core coordination by explicit message passing. Each instance is
big-lock-free *by construction* because it is single-threaded, so the
verification story survives onto multicore. The earlier framing of multikernel as
an "optional" final step (blueprint §16) is superseded — it is the baseline.

Implication for scheduling: runqueues are per-instance; cross-core load balancing
is an explicit message/migration, not a shared-structure steal. Producer/consumer
pairs for zero-copy should be kept on the same cluster (see [memory](memory.md)).

## Power-aware scheduling

Idle-state selection and DVFS are **policy** and live in user space, driven by the
power-management component (PSCI mechanism — see [drivers](drivers.md)). The
capacity-aware placement above is the hook for EAS-style energy-aware decisions
(ARM/Linux Energy Aware Scheduling is the reference). "Tickless idle" names the
*mechanism*; *which* idle state to enter is the policy's job.

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

## Sources

- SMP big-lock / verified-multicore status — seL4 FAQ; seL4 Summit 2022/2023
  multiprocessing slides; "The clustered multikernel" (UNSW); Barrelfish
  multikernel (SOSP 2009). See [ADR-0005](../adr/0005-smp-clustered-multikernel.md).
- Energy-aware scheduling — ARM/Linux EAS (kernel `sched-energy` docs).
- Revocation-vs-real-time — RTSS 2025 partitioning-kernel work (as above).

Full detail: blueprint §4.1. The choice of EEVDF over CFS-style heuristics is
informed by recent Linux scheduler work but not yet written up as its own ADR —
that's a gap (issue #3).
