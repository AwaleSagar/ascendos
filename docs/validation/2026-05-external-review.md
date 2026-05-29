# External review — 2026-05

An external, research-based validation pass was run against the blueprint and the
four ADRs, checking claims subsystem-by-subsystem against primary sources (ARM
docs, seL4 papers and specs, LWN, TCG, Linux kernel docs, academic papers). We
then independently re-verified its findings before acting on them. This page
records what it found, what we corrected in the review itself, and what changed
in the repo as a result. We keep this in the open because a design's credibility
comes partly from how it handles being checked.

## What held up

The core of the architecture verified cleanly against first-party sources:

- **Capability model** (untyped→retype, no kernel heap, mint/derive/badge/revoke)
  maps directly to seL4's verified design.
- **ARM exception-level mapping** (EL0 user / EL1 kernel / EL2 hypervisor /
  EL3 monitor; SVC/HVC/SMC routing; PSCI at EL3) is textbook-correct.
- **Scheduling-context capabilities** are a faithful adoption of seL4 MCS.
- **EEVDF** lineage (Stoica/Abdel-Wahab 1995 → Linux 6.6) is accurate.
- **Boot chain** (BootROM → BL1 → BL2 → BL31 → loader → kernel) matches the
  standard ARM TF-A flow; DICE/CDI usage matches the TCG spec.
- **User-space drivers, Nix/OSTree packaging, eBPF-class observability** all
  check out against their cited sources.
- **seL4 size** reference points confirmed: ~8,700 lines C + 600 asm (SOSP
  2009); ~10K (RISC-V) to ~12,000+ (AArch32) depending on architecture. Our
  ≤ 15 KLOC target is in the right ballpark but above seL4's verified size —
  achievable only with discipline, and the ring interface (ADR-0003) works
  against it.

## Errors we found *in the review* (and did not propagate)

A validation report is not automatically right. We caught two problems and
deliberately kept them out of the repo:

- **A fabricated CVE.** The review listed "CVE-2026-43174" among io_uring
  vulnerabilities. It does not exist. The real, citable io_uring CVEs are
  CVE-2021-3491, CVE-2023-1872, and CVE-2023-2598 — those are what
  [ADR-0003](../adr/0003-ring-based-syscall-interface.md) now cites.
- **A shaky performance number.** The review quoted "~1,830 cycles round-trip"
  for seL4 IPC. seL4's fastpath is platform-dependent and typically quoted far
  lower; we did **not** import this figure. We won't cite a cycle count until we
  measure one on a named target.

Lesson kept on the record: cite primary sources, and never launder a number you
haven't checked.

## What we changed in response

| Finding | Action | Where |
|---|---|---|
| 250 ms power-on target is unrealistic with firmware included | Split into S5→S8 (< 250 ms, our path) and S0→S8 (≤ 1 s, platform-dependent, reference SoC TBD) | [boot.md](../architecture/boot.md) |
| MTE presented as stronger than it is; async-mode overhead conflated with real protection | Added "MTE is a mitigation, not a guarantee" — probabilistic tags, TIKTAG speculative attack (S&P 2025), sync-vs-async cost, scarce hardware | [security.md](../architecture/security.md) |
| io_uring security record understated | Added real CVEs and the Google/Android/ChromeOS disable history; sharpened the verifiability-vs-rings conflict; added an explicit validation path | [ADR-0003](../adr/0003-ring-based-syscall-interface.md) |
| Capability revocation cost vs. real-time claim (missed by the review) | Documented the tension explicitly, citing RTSS 2025 work on bounding revocation cost | [scheduling.md](../architecture/scheduling.md) |
| "EEVDF-style" too vague | Noted what the eventual scheduler ADR must pin down | [scheduling.md](../architecture/scheduling.md), issue #3 |
| ARM memory ordering for rings unaddressed | Added STLR/LDAR analysis, per-entry cost on Cortex-A76, io_uring ordering bug history | [ADR-0003](../adr/0003-ring-based-syscall-interface.md) |
| CCA/RME hardware availability not discussed | Added availability note — no shipping silicon in 2026, design stays optional | [security.md](../architecture/security.md) |
| TCB 15 KLOC target lacks seL4 comparison | Added seL4 size context (8.7K–12.1K SLOC) and ring interface tension | [microkernel.md](../architecture/microkernel.md) |
| Architecture pages lack primary source citations | Added seL4 manuals, ARM ARM, io_uring manpage to capabilities, IPC, memory pages | [capabilities.md](../architecture/capabilities.md), [ipc.md](../architecture/ipc.md), [memory.md](../architecture/memory.md) |

## Risks the review missed

Worth recording, because they bear on our actual bets:

- **TIKTAG** — speculative-execution attacks defeat MTE tag protection in
  practice. The review validated that MTE exists and quoted its (async) overhead
  but missed that the protection is breakable. Now covered in
  [security.md](../architecture/security.md).
- **Revocation vs. real-time** — pure capabilities and bounded real-time are in
  tension; recursive revoke is long-running and hard to bound. Now in
  [scheduling.md](../architecture/scheduling.md).
- **Ring + weak memory + verifiability** — the review treated the ARM
  weak-memory issue as a documentation gap; we treat it as potentially in
  conflict with the small-verifiable-TCB thesis. See ADR-0003.

## Overall

The review's verdict — credible, research-grounded blueprint; two legitimate
flags (boot time, ring interface) — matches our own read after independent
checking. The repo is now more honest than before this pass, which is the point.
