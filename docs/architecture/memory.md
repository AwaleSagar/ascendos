# Memory

AscendOS uses the seL4-style **untyped → retype** model: all kernel objects are
allocated by user space from untyped memory capabilities, and the kernel keeps no
internal heap.

- **No kernel heap** → memory is fully accountable and there is no in-kernel OOM
  failure mode.
- **ARMv9 MTE** memory tagging used as defense-in-depth (not a guarantee — see
  [security](security.md)); **PAC/BTI** for control-flow integrity.
- Multi-level translation tables, **ASID-tagged TLB** entries to avoid full
  flushes on context switch, and contiguous-hint block mappings for TLB reach.
- **Demand paging is a user-space policy** — faults are delivered to a pager
  process by IPC; the kernel only routes them.

## ARM64 specifics worth pinning down now

The blueprint says "optional 52-bit VA"; the precise constraint matters:

- **ASID is 8-bit or 16-bit**, selected by `TCR_EL1.AS`. 16-bit ASIDs (65,536
  contexts) materially reduce ASID-recycle flushes on a busy microkernel system,
  so we target 16-bit where the implementation reports support.
- **52-bit virtual/physical addressing requires `FEAT_LPA2`** (and is only
  available with the 4 KB and 16 KB translation granules). It is not universally
  present, so 52-bit VA stays genuinely *optional* and feature-gated; the baseline
  is 48-bit VA with a 4 KB granule.
- **Granule choice (4/16/64 KB)** is a real trade-off (TLB reach vs. internal
  fragmentation) and should be a documented decision, not an accident.

## Open questions

- Tag (MTE) management across shared frames granted between processes needs a
  precise ownership story.
- Translation granule: 4 KB (compatibility) vs. 16 KB (better TLB reach, used by
  Apple platforms) — pick one as the default, justify it in an ADR.

## Sources

- `TCR_EL1` (ASID size `AS`, `FEAT_LPA2` for 52-bit) — Arm Architecture
  Reference / Arm Developer register docs.
- ARM64 page-table and ASID-tagged TLB behaviour — Linux Kernel Internals,
  "ARM64 Page Tables".

Full detail: blueprint §4.1.
