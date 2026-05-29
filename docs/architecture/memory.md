# Memory

AscendOS uses the seL4-style **untyped → retype** model: all kernel objects are
allocated by user space from untyped memory capabilities, and the kernel keeps no
internal heap.

- **No kernel heap** → memory is fully accountable and there is no in-kernel OOM
  failure mode.
- **ARMv9 MTE** memory tagging enforced; **PAC/BTI** for control-flow integrity.
- Multi-level translation tables, ASID recycling, optional 52-bit VA,
  block/contiguous mappings for TLB efficiency.
- **Demand paging is a user-space policy** — faults are delivered to a pager
  process by IPC; the kernel only routes them.

## Open questions

- Tag (MTE) management across shared frames granted between processes needs a
  precise ownership story.

Full detail: blueprint §4.1.
