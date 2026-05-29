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

## Sources

- seL4 Reference Manual: untyped memory and retype operations
  (<https://sel4.systems/Info/Docs/seL4-manual-latest.pdf>).
- ARM Architecture Reference Manual (ARM DDI 0487): MTE (Memory Tagging Extension),
  PAC (Pointer Authentication), BTI (Branch Target Identification).
- ARM developer guide: "Providing protection for complex software" (MTE/PAC/BTI overview)
  (<https://developer.arm.com/-/media/Arm%20Developer%20Community/PDF/Learn%20the%20Architecture/Providing%20protection%20for%20complex%20software.pdf>).
