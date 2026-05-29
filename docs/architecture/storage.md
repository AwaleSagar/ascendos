# Storage

> Status: outline. The storage stack is sketched but not designed in depth —
> it's a Phase 4 concern (see the roadmap).

Path: app → `vfsd` (VFS + per-process namespaces) → filesystem implementation →
`blockd` (buffer cache, I/O scheduler) → block driver → SMMU-guarded DMA → HW.

- Filesystem direction: copy-on-write + log-structured hybrid, per-block
  checksums, atomic snapshots (which feed package rollback).
- Optional per-file authenticated encryption with keys custodied by `secd`.
- Integrity for the immutable base image via a **dm-verity-style Merkle hash
  tree** — the root hash is established by the boot chain, and each block is
  verified on read, so tampering with the read-only rootfs is detected. This is
  the standard pattern for authenticated/immutable rootfs (AOSP, embedded Linux).
- All device DMA SMMU-scoped, as everywhere.

## A reality check on "write our own CoW filesystem"

The blueprint lists CoW + checksums + snapshots as a direction. The evidence
says this is the *hardest* subsystem to get right, not the easiest:

- ZFS and Btrfs are the reference CoW filesystems, and both took **many years** to
  reach production stability; crash-consistency, the free-space/ENOSPC problem,
  and fsck-without-a-fsck are genuinely hard. Practitioner write-ups (e.g. "The
  sorry state of CoW filesystems") document how long the tail of correctness is.
- Therefore the realistic plan is **not** "ship a novel CoW FS early." Order of
  operations: (1) a simple, correct, crash-consistent FS first (even
  log-structured-append with a checker) to get the system usable; (2) treat a
  full CoW/snapshotting FS as a later, separately-scoped effort, or port an
  existing design. Snapshots that `pkgd` rollback depends on can initially be
  provided at the **block/image (A/B) level** rather than requiring FS-level
  snapshots on day one.

Open question (now sharper): do we (a) port an existing CoW design, (b) build a
minimal correct FS first and defer CoW, or (c) lean on block-level A/B + verity
and delay an in-house FS entirely? Leaning (b)+(c) for early phases.

## Sources

- CoW filesystem design and maturity cost — ZFS/Btrfs internals write-ups;
  "The sorry state of CoW file systems" (Louwrentius); original Btrfs announce.
- Immutable rootfs integrity — dm-verity / Merkle-tree authenticated boot (TI
  AM62L Authenticated Boot guide; ProteanOS dm-verity guide).

Full detail: blueprint §8.
