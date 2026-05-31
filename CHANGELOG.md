# Changelog

All notable changes to this project are documented here. The format is based on
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/). This project is
pre-1.0 and the design may change without notice.

## [Unreleased]

### Added

- Initial architecture blueprint for the AscendOS ARM64 microkernel.
- Documentation scaffold (overview, architecture, ADRs, RFC process).
- Seed decision records ADR-0001 through ADR-0004.
- Project roadmap (Phase 0 onward).
- Community health files and contribution process.
- Validation record of the 2026-05 external review (`docs/validation/`).

### Changed

- Split the boot target into an AscendOS-owned path (S5→S8, < 250 ms) and a
  platform-dependent full power-on path (S0→S8, ≤ 1 s, reference SoC TBD), after
  review flagged the original single number as unrealistic.
- Qualified the MTE security claim: documented its probabilistic nature, the
  TIKTAG speculative attack, the sync-vs-async cost, and scarce hardware.
- Strengthened ADR-0003 with real io_uring CVEs, the Google/Android/ChromeOS
  disable history, the verifiability-vs-rings conflict, and a validation path.
- Documented the capability-revocation vs. real-time tension in scheduling.
- Added ARM memory-ordering analysis to ADR-0003 (STLR/LDAR, per-entry cost on
  Cortex-A76, io_uring ordering bug history).
- Added CCA/RME hardware availability note to security (no shipping silicon in
  2026, design stays optional).
- Added TCB size context to microkernel page (seL4 comparison, ring interface
  tension against 15 KLOC target).
- Added source citations to capabilities, IPC, memory, and other pages (seL4
  manuals, ARM Architecture Reference Manual, io_uring manpage).
- Comprehensive per-subsystem validation pass (`docs/validation/`): added a
  sourced IPC baseline (~986 cycles, seL4 fastpath); qualified 52-bit VA as
  requiring FEAT_LPA2 and specified ASID width; corrected "QUIC-ready L4" (QUIC
  runs over UDP) and added smoltcp/ixy.rs precedent; added a storage reality
  check (defer novel CoW; use A/B + dm-verity early); specified SMMU stage-1 /
  StreamID and GICv3 / ITS / LPI as the driver-isolation hardware baseline.
- Synced the master blueprint with the validated docs (boot split, MTE
  defense-in-depth, QUIC-over-UDP, 52-bit VA / FEAT_LPA2).
- Critical architecture review (`docs/validation/2026-05-critical-architecture-review.md`):
  nine evidence-backed findings prioritized by risk — SMP big-lock vs.
  verifiability (clustered multikernel), missing early-boot entropy/RNG,
  multi-server IPC overhead, unaddressed side-channels, missing power
  management, verified-Rust caveat, and more.
- Remediation plan (`docs/validation/2026-05-remediation-plan.md`) and applied
  all nine fixes:
  - **ADR-0005** — SMP as a clustered multikernel (supersedes the "optional
    multikernel" framing; preserves verifiability).
  - **ADR-0006** — first-class entropy/RNG subsystem (FEAT_RNG + persisted seed,
    blocks until seeded, gates key use).
  - **ADR-0007** — separate "memory safety" from "formal verification" as
    assurance goals; informs the open language decision.
  - Architecture pages updated: scheduling (SMP model, power-aware), security
    (entropy, transient-execution/timing channels + real-time tension), IPC
    (hot-path latency budget, NUMA/cache zero-copy), microkernel (square-law
    verification cost, memory-safe ≠ verified), drivers (PSCI power management),
    packaging (rollback via A/B + dm-verity, not FS snapshots).
  - ROADMAP and glossary updated for consistency.

Nothing is released yet. This section will be split into versioned entries once
there is something to version.
