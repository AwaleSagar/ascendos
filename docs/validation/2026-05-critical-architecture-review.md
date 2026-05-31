# Critical architecture review — 2026-05

A deliberately adversarial pass: what is *wrong*, *missing*, *unrealistic*, or
*risky* in the AscendOS blueprint. Every finding below is backed by a credible
source (ARM docs, OS research, kernel/OS conference material, or a proven
open-source OS). Nothing here is speculation; where something is a judgement
call it is labelled as such.

This complements the two earlier passes
([external review](2026-05-external-review.md),
[comprehensive pass](2026-05-comprehensive-pass.md)) — those validated what the
blueprint *claims*; this one attacks it.

## How to read the table

- **Severity** — impact if left unaddressed: 🔴 high · 🟠 medium · 🟡 low.
- **Effort** — rough cost to address: ⛰️ large · 🪜 medium · 🔧 small.
- Findings are ordered **high-severity-first**, then by lower effort within a
  severity band (fix the cheap high-impact things first).

---

## F1 — SMP scalability: the big-lock reality 🔴 / ⛰️

**Claim under review.** §16 lists a scale path "single core → SMP → NUMA-aware →
optional multikernel," and the scheduler page promises per-CPU runqueues with
work-stealing — implying smooth multicore scaling.

**Evidence.** The closest verified peer, seL4, **does not** scale that way. Per
the seL4 FAQ: *"The SMP kernel uses a big-lock approach"* and *"verification of
the multicore kernel is in progress for static multikernel configurations."*
The research direction the seL4 team actually pursues is the **clustered
multikernel** (UNSW thesis, "The clustered multikernel"), precisely because a
shared, concurrent, *verifiable* kernel is so hard. Their 2022/2023 Summit talks
state the constraints bluntly: "No concurrency in the kernel. No changes to
verified C code."

**Impact.** AscendOS wants both a small *verifiable* TCB (the whole thesis) and
fine-grained SMP scalability. Those are in tension: a big lock is verifiable but
caps multicore throughput; a fine-grained concurrent kernel scales but is, today,
beyond tractable verification. Presenting SMP as a smooth later step understates
that this is an open research problem even for the best-funded verified kernel.

**Recommendation.** Adopt the **multikernel-per-core / clustered** model as the
*primary* SMP story, not an "optional" footnote — i.e. replicate kernel state
per core/cluster and communicate by message-passing (Barrelfish multikernel,
SOSP 2009), rather than promising a shared concurrent kernel. Reframe §16 and
the scheduler page accordingly, and add an ADR recording that SMP = multikernel,
chosen for verifiability.

*Sources: seL4 FAQ; "The clustered multikernel" (UNSW); seL4 Summit 2022/2023
multiprocessing slides; Barrelfish multikernel (SOSP 2009).*

---

## F2 — Multi-server IPC overhead compounds 🔴 / 🪜

**Claim under review.** The design routes everything through user-space servers
(vfsd → blockd → driver; netstackd → driver), betting that ring-based IPC keeps
this cheap.

**Evidence.** The foundational study — Härtig et al., *"The Performance of
µ-Kernel-Based Systems"* (SOSP 1997) — showed system-call and context-switch
overhead "counts heavily" once OS personality work is split across user-mode
servers, and that naive decomposition cost the original Mach-based systems dearly.
A single logical operation (e.g. an open-read on a file) can cross *several*
protection domains, each adding a context switch. L4Re's own published per-IPC
numbers exist precisely because this cost is real and must be budgeted.

**Impact.** A file read in AscendOS may traverse app → vfsd → blockd → NVMe
driver and back — potentially 6–8 domain crossings. Even at seL4's ~986-cycle
fastpath, a deep call chain multiplies that, and the ring interface only helps
when calls *batch*; a latency-bound single request still pays per-hop. This is
the classic microkernel tax and the blueprint doesn't budget for it.

**Recommendation.** (1) Add a **latency budget** to the IPC page that counts
worst-case domain crossings for the hot paths (file read, packet RX) and states
a target. (2) Design server **co-location / fast-path fusion** where justified
(e.g. vfsd+blockd in one address space initially), and revisit only if measured.
(3) Make "minimise domain crossings on the hot path" an explicit design rule.

*Sources: Härtig et al., SOSP 1997 (MIT 6.828 / Cornell CS6410 course copies);
L4Re Performance Overview.*

---

## F3 — Missing subsystem: early-boot entropy / RNG 🔴 / 🪜

**Claim under review.** The security framework covers attestation, capabilities,
MTE/PAC; the boot chain covers measurement. **Nothing** in the blueprint provides
randomness or an entropy source.

**Evidence.** This is a well-documented real-world failure class. The CMU/HotOS
paper *"Recommendations for Randomness in the Operating System"* and Red Hat's
RNG guidance both document that systems generating keys **early after boot** —
before enough entropy is collected — produce predictable keys; the famous
large-scale study found embedded/headless devices generating weak SSH/TLS keys
for exactly this reason. A CLI-only, headless, embedded-leaning OS (no mouse, no
GUI, sparse interrupt entropy) is the *worst case* for entropy starvation.

**Impact.** Any security property that needs unpredictability — attestation
nonces, capability badge randomization, TLS in netstackd, ASLR if added, package
signature challenge — is compromised if the RNG is weak at first boot. This
silently undermines the entire security framework the blueprint is proud of.

**Recommendation.** Add an **entropy subsystem** as a named component: seed from
hardware RNG where present (ARMv8.5 **FEAT_RNG** / `RNDR`), the DICE/boot
measurement secrets, and a persisted seed file across boots; gate
security-critical key generation on a "RNG ready" signal (as Linux's
`getrandom()` blocks until initialized). This is a small spec addition with
outsized security value.

*Sources: "Recommendations for Randomness in the OS" (HotOS/CMU); Red Hat RNG
guidance; ARM FEAT_RNG / RNDR architecture.*

---

## F4 — Side-channel / speculative attacks unaddressed 🟠 / 🪜

**Claim under review.** Security rests on isolation (capabilities, SMMU,
per-process address spaces). Spectre-class transient-execution and timing
channels are not mentioned.

**Evidence.** Spectre is a *class*, not a single bug — "there is no single
mitigation" (FAU EuroSec 2021, *"The Price of Meltdown and Spectre"*). Isolation
boundaries do **not** stop transient-execution leaks by themselves, and
mitigations carry real cost — measurable energy and latency overhead, and
specifically a **real-time** cost (Ramsauer/Kiszka/Mauerer, "Spectre and Meltdown
vs. Real-Time"). A microkernel's frequent domain crossings can themselves form a
timing channel.

**Impact.** AscendOS markets strong isolation; an unaddressed Spectre variant
lets a confined process read across the boundary the capability model claims to
enforce. Worse, the blueprint's *other* goal — hard real-time via MCS — is
directly harmed by the mitigations (flushes on context switch). These two goals
collide, undocumented.

**Recommendation.** Add a "transient execution & timing channels" section to the
threat model (issue #2): state which variants are in scope, plan for context-
switch flushing / branch-predictor controls on ARM, and **acknowledge the
real-time tension** (mitigations vs. bounded latency) as an explicit trade-off
rather than discovering it later.

*Sources: meltdownattack.com; "The Price of Meltdown and Spectre" (FAU EuroSec
2021); "Spectre and Meltdown vs. Real-Time" (Linux Foundation summit).*

---

## F5 — Missing subsystem: power management 🟠 / 🪜

**Claim under review.** The HAL lists timers, GIC, SMMU. There is **no** power
management: no CPU idle states, no DVFS, no suspend/resume, no CPU hotplug
beyond bring-up. For an ARM64 OS targeting "minimal footprint" and embedded-class
hardware, this is a notable gap.

**Evidence.** ARM standardizes this via **PSCI** (Power State Coordination
Interface): `cpuidle`/hotplug, suspend-to-RAM, and suspend-to-idle (S2Idle) are
all PSCI-mediated on ARMv8-A (ARM developer docs; TI AM62L S2Idle/PSCI guide;
Linaro "Linux Kernel Power Management"). The blueprint already invokes PSCI at
EL3 for boot, but never for runtime power.

**Impact.** Without idle states and DVFS, an AscendOS device burns power at full
tilt — disqualifying for battery/embedded use and contradicting the
minimal-footprint positioning. Retrofitting power management touches the
scheduler (tickless idle is claimed but idle-state *selection* isn't),
interrupts, and drivers, so adding it late is expensive.

**Recommendation.** Add a **power-management component** that uses PSCI for CPU
idle/hotplug/suspend and a policy (in user space, consistent with the
mechanism/policy split) for DVFS and idle-state selection. Tie it to the
EEVDF/capacity-aware scheduler (EAS-style energy awareness — ARM/Linux EAS is the
reference). Mark depth as Phase 3/4 but name the component now.

*Sources: ARM PSCI specification; ARM "cpuidle (hotplug)"; TI S2Idle/PSCI
integration; Linaro power-management overview.*

---

## F6 — Maintainability: verified-kernel-in-Rust is largely unproven 🟠 / ⛰️

**Claim under review.** The design implies a memory-safe kernel language (Rust
leaning) *and* a verifiable ≤ 15 KLOC TCB.

**Evidence.** seL4's verification is of **C**, with a mature Isabelle/HOL proof
chain built over a decade; there is no comparably mature, end-to-end *functional-
correctness* proof stack for a Rust kernel today. Rust gives memory safety, not
functional correctness, and kernels need `unsafe` exactly where verification
matters most (MMIO, page tables, the ring). Rust's own 2026 "challenges" writeup
and practitioner accounts note that designing *around* the borrow checker for
low-level data structures (intrusive lists, shared kernel state) is the real
cost, not the checker itself.

**Impact.** The blueprint conflates "memory-safe language" with "verifiable." If
the goal is seL4-level assurance, choosing Rust may mean *more* verification R&D,
not less — a maintainability and timeline risk that isn't acknowledged.

**Recommendation.** In the (still-open) language ADR, separate two goals
explicitly: **memory safety** (Rust delivers) vs. **functional verification**
(Rust does *not* yet deliver at seL4's level). Decide which assurance level
AscendOS actually targets. If full verification is the goal, weigh C-plus-proof
vs. Rust-plus-emerging-tooling honestly; if memory-safety-plus-testing is enough,
say so and drop the "verifiable" framing to "auditable."

*Sources: seL4 proofs page (C / Isabelle); Rust blog "What we heard about Rust's
challenges" (2026); Redox/Theseus as the state of Rust OS practice.*

---

## F7 — "Verifiable TCB" vs. the ring interface, restated as a gap 🟠 / 🪜

**Claim under review.** Already partly captured in
[ADR-0003](../adr/0003-ring-based-syscall-interface.md), but worth listing as a
standalone architecture risk: verification cost grows roughly with the **square**
of code size (microkerneldude / seL4 team), so every KLOC the ring adds to the
TCB is super-linear proof cost.

**Impact.** The ring interface and the 15 KLOC verifiable target are not just in
mild tension — if verification is a real goal, the ring may be unaffordable to
prove. This sharpens F6.

**Recommendation.** Keep the ADR-0003 validation gate, and add the square-law
verification-cost note to the microkernel TCB discussion so the trade-off is
visible where the size budget lives.

*Source: microkerneldude.org (Heiser), verification cost ∝ ~code-size².*

---

## F8 — Storage rollback depends on undecided FS (consistency) 🟡 / 🪜

**Claim under review.** `pkgd` atomic rollback depends on FS snapshots, but the
filesystem is explicitly undecided and (per the comprehensive pass) a novel CoW
FS is deferred.

**Evidence.** CoW correctness is a multi-year effort (ZFS/Btrfs history). If
rollback is specified against a capability the FS doesn't yet provide, the
packaging guarantee is unbacked.

**Impact.** A headline feature (atomic rollback) rests on an unbuilt dependency.

**Recommendation.** Already partially addressed — bind rollback to **block-level
A/B + dm-verity** for early phases (does not need FS snapshots), and state that
FS-level snapshots are an enhancement, not a prerequisite. Make the dependency
direction explicit in the packaging page.

*Sources: ZFS/Btrfs maturity write-ups; dm-verity authenticated boot (already
cited in storage.md).*

---

## F9 — No clear NUMA / cache-coherency story for shared-frame zero-copy 🟡 / 🪜

**Claim under review.** Zero-copy via shared frames is central to IPC,
networking, and storage performance.

**Evidence.** On multicore ARM, shared-frame performance depends on cache
shareability domains (Inner/Outer Shareable) and, on larger systems, NUMA
placement; seL4's SMP investigations explicitly study "impact of replicated data
on shared caches" (seL4 Summit 2023). Cross-core shared frames without attention
to placement can thrash caches.

**Impact.** The zero-copy win can evaporate (or invert) across cores/clusters if
frames bounce between caches — undermining the performance argument that
justifies the ring complexity.

**Recommendation.** Add a note to the IPC/memory pages that shared-frame
placement must be cache/cluster-aware, consistent with the F1 multikernel
direction (keep producer/consumer on the same cluster where possible). Defer
detail, but name the concern.

*Source: seL4 Summit 2023, "Are efficient SMP VMs possible on verifiable seL4?"*

---

## Summary table (prioritized)

| # | Finding | Severity | Effort | Core recommendation |
|---|---|---|---|---|
| F1 | SMP = big-lock vs. verifiability | 🔴 | ⛰️ | Make clustered multikernel the primary SMP model |
| F3 | Missing early-boot entropy/RNG | 🔴 | 🪜 | Add entropy subsystem (FEAT_RNG + seed + gate) |
| F2 | Multi-server IPC overhead compounds | 🔴 | 🪜 | Budget hot-path domain crossings; allow fusion |
| F4 | Side-channel / Spectre unaddressed | 🟠 | 🪜 | Add to threat model; note real-time tension |
| F5 | Missing power management | 🟠 | 🪜 | PSCI cpuidle/suspend + user-space DVFS policy |
| F7 | Ring vs. verifiable TCB (square-law cost) | 🟠 | 🪜 | Add verification-cost note to TCB budget |
| F6 | Verified-Rust-kernel unproven | 🟠 | ⛰️ | Separate "memory-safe" from "verified" in lang ADR |
| F8 | Rollback depends on undecided FS | 🟡 | 🪜 | Bind rollback to A/B + dm-verity early |
| F9 | No NUMA/cache story for zero-copy | 🟡 | 🪜 | Cache/cluster-aware shared-frame placement |

## Verdict

The blueprint remains conceptually sound — none of these is a fatal flaw, and the
fixes are mostly additive. The three high-severity items share a theme: the
blueprint is strongest on the *single-core, security* story and weakest where
**multicore reality** and **operational subsystems** (entropy, power) meet the
verifiability thesis. The most valuable cheap win is **F3 (entropy)**; the most
important reframing is **F1 (SMP as multikernel)**.
