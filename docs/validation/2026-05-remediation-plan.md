# Remediation plan — 2026-05

This plan turns the [critical architecture review](2026-05-critical-architecture-review.md)
into concrete, prioritized blueprint changes. Each item validates the proposed
fix against credible sources, names the documents/diagrams to change, and records
a change-log entry (original design → issue → evidence → revised design →
benefits → trade-offs).

Work is applied **incrementally**, highest priority first. Priority combines
severity, architectural impact, implementation risk, and evidence strength.

## Priority order

| Order | Finding | Why this rank | Status |
|---|---|---|---|
| P1 | F3 — entropy/RNG subsystem | High severity, low effort, strong evidence, security-critical | applied |
| P2 | F1 — SMP = clustered multikernel | High severity, high architectural impact, strong evidence | applied |
| P3 | F2 — multi-server IPC budget | High severity, medium effort, foundational evidence | applied |
| P4 | F5 — power management (PSCI) | Medium severity, missing subsystem, standard fix | applied |
| P5 | F4 — side-channels in threat model | Medium severity, medium effort | applied |
| P6 | F7 — ring vs. verification cost | Medium, sharpens existing ADR-0003 | applied |
| P7 | F6 — memory-safe ≠ verified | Medium, informs open language ADR | applied |
| P8 | F8 — rollback ↔ FS dependency | Low, already half-addressed | applied |
| P9 | F9 — NUMA/cache zero-copy | Low, note-only | applied |

New decision records created: **ADR-0005** (SMP = clustered multikernel),
**ADR-0006** (entropy/RNG subsystem), **ADR-0007** (memory-safety vs.
verification assurance levels).

---

## Change log

### CL-1 — Entropy / RNG subsystem (F3)

- **Original design.** No randomness source anywhere in the blueprint. Security
  framework assumed unpredictability (attestation nonces, key generation, badge
  randomization) without providing it.
- **Issue.** A headless, CLI-only, embedded-leaning OS has the *least* ambient
  entropy (no GUI input, sparse interrupts) — the documented worst case for weak
  early-boot keys.
- **Evidence.** "Recommendations for Randomness in the Operating System"
  (HotOS/CMU); Red Hat RNG guidance (devices that generate keys early after boot
  produce predictable keys); ARMv8.5 **FEAT_RNG** / `RNDR` provides a hardware
  entropy source.
- **Revised design.** New `entropyd` (or kernel-assisted CSPRNG) component:
  seeds from FEAT_RNG `RNDR` where present, DICE/boot measurement secrets, and a
  persisted seed file across boots; exposes a `getrandom`-style interface that
  **blocks until first good seed**; security-critical key generation gates on a
  "RNG ready" signal.
- **Benefits.** Closes a silent compromise of every security property that needs
  unpredictability; aligns with proven Linux `getrandom()` semantics.
- **Trade-offs.** Boot path must reach "RNG ready" before key use — a small
  ordering constraint on early services; persisted-seed file needs integrity
  (covered by dm-verity / secd).
- **Files changed.** `security.md` (new section), service mesh on the blueprint
  (`§7`, add `entropyd`), new **ADR-0006**, glossary (FEAT_RNG, CSPRNG).

### CL-2 — SMP as clustered multikernel (F1)

- **Original design.** §16 framed SMP as a smooth step ("single core → SMP →
  NUMA-aware → optional multikernel"); scheduler promised per-CPU runqueues with
  work-stealing, implying a shared concurrent kernel.
- **Issue.** A shared, concurrent, *verifiable* kernel is an open research
  problem; promising smooth SMP scaling understates the difficulty and conflicts
  with the small-verifiable-TCB thesis.
- **Evidence.** seL4 FAQ: "The SMP kernel uses a big-lock approach … verification
  of the multicore kernel is in progress for static multikernel configurations."
  UNSW "The clustered multikernel." Barrelfish multikernel (SOSP 2009). seL4
  Summit 2022/2023: "No concurrency in the kernel."
- **Revised design.** Make the **clustered/per-core multikernel the primary SMP
  model**: replicate kernel state per core (or per cluster), communicate by
  message passing, keep each kernel instance single-threaded (big-lock-free *by
  construction*, hence verifiable). A single shared concurrent kernel is
  explicitly **rejected**.
- **Benefits.** Preserves verifiability while scaling; matches the only credible
  precedent for verified multicore.
- **Trade-offs.** Cross-core operations become explicit messages (higher latency
  than shared memory); state replication costs memory and adds a consistency
  protocol; NUMA/cache placement now matters (see CL-9).
- **Files changed.** `scheduling.md`, blueprint `§16` (extensibility/scale),
  blueprint `§4.1`, new **ADR-0005**.

### CL-3 — Multi-server IPC latency budget (F2)

- **Original design.** Everything routed through user-space servers
  (app→vfsd→blockd→driver), assuming ring IPC keeps it cheap.
- **Issue.** A single logical operation crosses several protection domains; each
  crossing costs a context switch. Rings help only when calls batch; latency-
  bound single requests pay per hop.
- **Evidence.** Härtig et al., "The Performance of µ-Kernel-Based Systems" (SOSP
  1997) — system-call/context-switch overhead "counts heavily" under server
  decomposition. L4Re publishes per-IPC numbers precisely because the cost is
  real.
- **Revised design.** Add an explicit **hot-path latency budget**: count
  worst-case domain crossings for file-read and packet-RX, state a target in
  crossings and cycles; permit **server fusion** (e.g. vfsd+blockd co-located)
  where measured need justifies it; make "minimise hot-path crossings" a design
  rule.
- **Benefits.** Makes the microkernel tax visible and bounded rather than
  discovered late.
- **Trade-offs.** Fusion trades a little isolation for latency; must be justified
  case-by-case, not by default.
- **Files changed.** `ipc.md` (latency-budget section), blueprint `§6`.

### CL-4 — Power management via PSCI (F5)

- **Original design.** HAL listed timers/GIC/SMMU; PSCI used only for boot. No
  runtime CPU idle, DVFS, suspend, or hotplug.
- **Issue.** Full-tilt power draw disqualifies battery/embedded use and
  contradicts the minimal-footprint positioning.
- **Evidence.** ARM **PSCI** specification standardizes cpuidle/hotplug,
  suspend-to-RAM and S2Idle on ARMv8-A (ARM developer docs; TI AM62L S2Idle/PSCI
  guide; Linaro PM overview). ARM/Linux **EAS** is the reference for energy-aware
  placement.
- **Revised design.** New **power-management component**: PSCI for CPU
  idle/hotplug/suspend (mechanism), with idle-state selection and DVFS as a
  **user-space policy** (consistent with the mechanism/policy split), tied to the
  capacity-aware scheduler (EAS-style).
- **Benefits.** Enables real battery/embedded operation; reuses an ARM standard.
- **Trade-offs.** Touches scheduler (idle-state selection vs. tickless idle),
  IRQs, and drivers (device suspend/resume) — depth deferred to Phase 3/4 but the
  component is named now.
- **Files changed.** `drivers.md`/HAL note, scheduler note, blueprint `§1`/`§14`,
  ROADMAP (Phase 3/4 exit criteria).

### CL-5 — Side-channels in the threat model (F4)

- **Original design.** Security rested on isolation; transient-execution and
  timing channels unmentioned.
- **Issue.** Isolation boundaries don't stop Spectre-class leaks; mitigations
  carry real-time cost that collides with the MCS hard-real-time goal.
- **Evidence.** meltdownattack.com; "The Price of Meltdown and Spectre" (FAU
  EuroSec 2021); "Spectre and Meltdown vs. Real-Time" (Linux Foundation).
- **Revised design.** Add a "transient execution & timing channels" subsection:
  scope the variants, plan context-switch flushing / branch-predictor controls on
  ARM, and **explicitly record the mitigation-vs-real-time tension**.
- **Benefits.** Removes a blind spot that would otherwise void the isolation
  claim.
- **Trade-offs.** Mitigations cost cycles and complicate the real-time bound — an
  acknowledged, not hidden, conflict.
- **Files changed.** `security.md` (threat-model subsection), `scheduling.md`
  cross-reference.

### CL-6 — Ring vs. verification cost, restated (F7)

- **Original design.** ADR-0003 noted ring complexity vs. small TCB.
- **Issue.** Verification cost grows ~with the square of code size, so ring KLOC
  is super-linear proof cost — sharper than "some tension."
- **Evidence.** microkerneldude (Heiser): verification cost ∝ ~code-size².
- **Revised design.** Add the square-law note to the microkernel TCB-size
  discussion and cross-link ADR-0003.
- **Benefits.** Puts the cost where the size budget lives.
- **Trade-offs.** None (documentation precision).
- **Files changed.** `microkernel.md`, ADR-0003 cross-ref.

### CL-7 — Memory-safe ≠ verified (F6)

- **Original design.** Implied Rust-leaning memory-safe kernel *and* verifiable
  TCB, conflating the two.
- **Issue.** Rust delivers memory safety, not functional correctness; no mature
  end-to-end Rust functional-verification stack exists at seL4's level.
- **Evidence.** seL4 proofs are of C in Isabelle/HOL; Rust blog "What we heard
  about Rust's challenges" (2026); Redox/Theseus as the state of practice.
- **Revised design.** New **ADR-0007** separating assurance levels: **memory
  safety** (Rust delivers) vs. **functional verification** (Rust does not yet).
  Pick the target; if full verification is the goal, weigh C+proof vs.
  Rust+emerging-tooling; if not, reframe "verifiable" → "auditable."
- **Benefits.** Removes a misleading conflation; sets honest expectations.
- **Trade-offs.** May concede that seL4-level assurance and Rust are not yet both
  achievable — an honest limitation.
- **Files changed.** new **ADR-0007**, `microkernel.md`, `faq.md`.

### CL-8 — Rollback ↔ filesystem dependency (F8)

- **Original design.** `pkgd` atomic rollback depended on FS snapshots, but the
  FS is undecided / novel-CoW deferred.
- **Issue.** A headline feature rested on an unbuilt dependency.
- **Evidence.** ZFS/Btrfs maturity history; dm-verity authenticated boot.
- **Revised design.** Bind early-phase rollback to **block-level A/B + dm-verity**
  (no FS snapshots needed); FS-level snapshots become an enhancement.
- **Benefits.** Rollback works before a CoW FS exists.
- **Trade-offs.** A/B uses more storage than FS snapshots.
- **Files changed.** `packaging.md`, `storage.md` (dependency direction).

### CL-9 — NUMA/cache-aware zero-copy (F9)

- **Original design.** Zero-copy shared frames assumed uniformly cheap.
- **Issue.** Cross-core shared frames can thrash caches; placement matters,
  especially under the CL-2 multikernel model.
- **Evidence.** seL4 Summit 2023, "Are efficient SMP VMs possible on verifiable
  seL4?" (impact of replicated data on shared caches).
- **Revised design.** Note that shared-frame producers/consumers should be kept
  on the same cluster where possible; Inner/Outer-Shareable choice matters.
- **Benefits.** Protects the zero-copy performance argument under SMP.
- **Trade-offs.** Placement constraints on scheduling (consistent with CL-2).
- **Files changed.** `ipc.md`/`memory.md` note.

---

## Consistency pass

After applying CL-1…CL-9, the following are kept consistent:

- **Diagrams.** Blueprint `§1` (band diagram) and `§7` (service mesh) gain
  `entropyd` and a power-management component; `§16` scale diagram reframed to
  multikernel.
- **ADR index.** README table updated with ADR-0005/0006/0007.
- **mkdocs nav.** New ADRs and this plan added.
- **CHANGELOG.** One "Changed" block summarising the remediation.
- **ROADMAP.** Entropy and power-management components named in the relevant
  phases.
