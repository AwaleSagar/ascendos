# Packaging

> Status: outline. Phase 4 concern.

A content-addressed, immutable package store (Nix/OSTree lineage) with atomic,
transactional activation and instant rollback:

- Packages are hash-named and immutable; no in-place mutation.
- Activation: stage new generation → `secd` verifies signatures → atomic root
  swap; failure rolls back.
- Declarative: desired system state is a manifest that `pkgd` reconciles.
- A/B base image for safe OS upgrades.

## Rollback does not depend on a filesystem we haven't built

The "instant rollback" guarantee originally read as if it required filesystem
snapshots — but the filesystem is undecided and a novel CoW design is explicitly
deferred ([storage](storage.md)). Resting a headline feature on an unbuilt
dependency is a risk, so we invert it:

- **Early phases:** rollback is provided at the **block/image level via A/B
  partitions + dm-verity** integrity. This needs no FS-level snapshots — the
  proven pattern for immutable/atomic OS images (OSTree/ABRoot, AOSP A/B).
- **Later (enhancement):** FS-level snapshots, *if* a CoW filesystem lands, become
  a finer-grained addition — not a prerequisite.

So the dependency direction is: packaging rollback → A/B + verity (firm) →
optionally FS snapshots (later). Packaging is not blocked on the filesystem
decision.

## Sources

- Immutable/atomic store and rollback — Nix/OSTree design; ABRoot/AOSP A/B.
- Image integrity — dm-verity authenticated boot (see [storage](storage.md)).

Full detail: blueprint §12.
