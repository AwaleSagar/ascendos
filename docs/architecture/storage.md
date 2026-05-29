# Storage

> Status: outline. The storage stack is sketched but not designed in depth —
> it's a Phase 4 concern (see the roadmap).

Path: app → `vfsd` (VFS + per-process namespaces) → filesystem implementation →
`blockd` (buffer cache, I/O scheduler) → block driver → SMMU-guarded DMA → HW.

- Filesystem direction: copy-on-write + log-structured hybrid, per-block
  checksums, atomic snapshots (which feed package rollback).
- Optional per-file authenticated encryption with keys custodied by `secd`.
- All device DMA SMMU-scoped, as everywhere.

Open question: do we ship our own filesystem or port an existing CoW design?
Undecided.

Full detail: blueprint §8.
