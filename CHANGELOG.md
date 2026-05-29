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
<<<<<<< HEAD
- Comprehensive per-subsystem validation pass (`docs/validation/`): added a
  sourced IPC baseline (~986 cycles, seL4 fastpath); qualified 52-bit VA as
  requiring FEAT_LPA2 and specified ASID width; corrected "QUIC-ready L4" (QUIC
  runs over UDP) and added smoltcp/ixy.rs precedent; added a storage reality
  check (defer novel CoW; use A/B + dm-verity early); specified SMMU stage-1 /
  StreamID and GICv3 / ITS / LPI as the driver-isolation hardware baseline.
=======
- Added ARM memory-ordering analysis to ADR-0003 (STLR/LDAR, per-entry cost on
  Cortex-A76, io_uring ordering bug history).
- Added CCA/RME hardware availability note to security (no shipping silicon in
  2026, design stays optional).
- Added TCB size context to microkernel page (seL4 comparison, ring interface
  tension against 15 KLOC target).
- Added source citations to capabilities, IPC, and memory pages (seL4 manuals,
  ARM Architecture Reference Manual, io_uring manpage).
>>>>>>> origin/main

Nothing is released yet. This section will be split into versioned entries once
there is something to version.
