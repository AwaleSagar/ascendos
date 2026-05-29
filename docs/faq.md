# FAQ

**Can I run AscendOS?**
No. There is no code yet. This repository is a design blueprint and the
reasoning behind it. If and when there's something to boot, the
[roadmap](https://github.com/AwaleSagar/ascendos/blob/main/ROADMAP.md) will say
so.

**Is this another Linux?**
No. It's a from-scratch microkernel design with no POSIX or binary compatibility
goal. It borrows ideas from seL4, Genode, Redox, and Fuchsia — see the README's
prior-art section.

**Why no GUI?**
Deliberate and permanent. A GUI stack is a large attack and maintenance surface,
and the project's value is in the kernel. See [Non-goals](overview/non-goals.md).

**What language will it be written in?**
Not decided. Leaning toward a memory-safe systems language (Rust is the obvious
candidate), but that's a Phase 1 decision and will get its own ADR.

**Why ARM64 only?**
Portability abstraction has a real cost, and ARM64 is where the isolation
hardware we want (SMMU, MTE, PAC/BTI, CCA) lives. Multi-arch is an explicit
non-goal for now.

**Is the "< 250 ms boot" / "< 32 MB RAM" real?**
No — those are *targets*, not measurements. Nothing has been built or measured.
Wherever you see a number in these docs, read it as an intention.

**How can I help?**
Right now, by critiquing the design — especially
[ADR-0003](adr/0003-ring-based-syscall-interface.md). See
[CONTRIBUTING](https://github.com/AwaleSagar/ascendos/blob/main/.github/CONTRIBUTING.md).

**Who's behind this?**
One maintainer, [@AwaleSagar](https://github.com/AwaleSagar), for now. See
[GOVERNANCE](https://github.com/AwaleSagar/ascendos/blob/main/.github/GOVERNANCE.md).
