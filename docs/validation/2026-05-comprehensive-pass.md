# Comprehensive validation pass — 2026-05

A second, wider validation pass after the [external review](2026-05-external-review.md):
every subsystem, interface, and assumption checked against credible sources only
(Arm architecture docs and registers, OS research papers, kernel/OS conference
talks, and proven open-source implementations — seL4, Redox, Barrelfish, Plan 9,
DPDK, ZFS/Btrfs, dm-verity). The rule for this pass: **change something only where
a credible source supports the change**, and record the rationale.

## Confirmed sound (no change needed)

| Subsystem | Verified against | Verdict |
|---|---|---|
| Untyped→retype, no kernel heap, capabilities | seL4 manual & SOSP 2009 | ✅ accurate |
| EL0–EL3 model, SVC/HVC/SMC, PSCI@EL3 | Arm AArch64 Exception Model docs | ✅ accurate |
| Scheduling-context capabilities | seL4 MCS docs + Lyons et al. | ✅ accurate |
| User-space drivers + SMMU isolation | Arm SMMU docs; Fuchsia DFv2; ixy.rs/Redox | ✅ accurate |
| DICE/measured boot, TF-A flow | TCG DICE 1.1; TF-A measured-boot docs | ✅ accurate |
| Content-addressed atomic packaging | Nix/OSTree docs | ✅ accurate |
| eBPF-class sandbox over native modules | ebpf.io; eBPF verifier CVE history | ✅ accurate |
| Plan 9 per-process namespaces | "The Use of Name Spaces in Plan 9" | ✅ accurate |
| Multikernel scale-out option | Barrelfish multikernel SOSP 2009 | ✅ sound as an *option* |

## Changes made (each evidence-backed)

| Change | Rationale | Source |
|---|---|---|
| Added a real IPC baseline (~986 cycles seL4 fastpath round-trip) to [ipc.md](../architecture/ipc.md); explicitly retired the unsourced "~1,830 cycles" | The ring interface must beat a *known* number; gives ADR-0003 a concrete bar | LionsOS, arXiv 2501.06234; seL4 "Correct, Fast, Maintainable" |
| [memory.md](../architecture/memory.md): qualified "52-bit VA" as requiring **FEAT_LPA2** (4 KB/16 KB granules); specified ASID is **8/16-bit** via `TCR_EL1.AS`; added granule-choice as a decision | "optional 52-bit VA" was underspecified; 52-bit isn't universal and the ASID width has real flush-rate consequences | Arm `TCR_EL1` register docs; Linux Kernel Internals ARM64 page tables |
| [networking.md](../architecture/networking.md): corrected "QUIC-ready L4" → QUIC runs **over UDP** (not an L4 peer); added smoltcp/ixy.rs precedent and a lean toward **porting smoltcp** | "QUIC as L4" is technically wrong; reusing a proven `no_std` stack de-risks a hard subsystem | RFC 9000; Redox 0.3.5 notes; "Porting ixy.rs to Redox" (TUM); DPDK Z-stack |
| [storage.md](../architecture/storage.md): added a reality check — defer a novel CoW FS; do a simple correct FS first + **block-level A/B** and **dm-verity** Merkle integrity early | ZFS/Btrfs evidence shows CoW correctness is a multi-year tail; rollback doesn't need FS snapshots on day one | ZFS/Btrfs internals; "sorry state of CoW filesystems"; dm-verity authenticated-boot guides |
| [drivers.md](../architecture/drivers.md): specified **SMMU stage-1 + StreamID** as the DMA-confinement mechanism and **GICv3/ITS/LPI + DeviceID** as the interrupt baseline; flagged StreamID/DeviceID table ownership as security-critical | The isolation guarantee depends on these exact hardware details; "SMMU" alone was too vague | Arm "What an SMMU does"; openEuler SMMU intro; Arm GICv3/v4 LPI docs |

## Deliberately *not* changed

- **Multikernel** stays an optional future direction, not a baseline. Barrelfish
  validates the model for scalability, but state replication adds complexity a
  single small kernel doesn't need yet. Correct to keep it in "future
  extensibility," not the core.
- **init/svcd model** (dependency-ordered parallel start, supervision, socket/
  endpoint activation) matches s6/systemd-class practice; no change warranted.
- **Implementation language** remains undecided (Rust leaning). Redox and Theseus
  validate Rust for OS work, but this is a Phase 1 ADR, not something to fix now.

## Net

The architecture is internally consistent and well-grounded. This pass tightened
five subsystem pages with specific, cited hardware/protocol facts and one
honest scope-reduction (storage). No fundamental design change was required —
the corrections were precision and realism, which is what a pre-implementation
blueprint most needs.
