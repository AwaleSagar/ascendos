# Packaging

> Status: outline. Phase 4 concern.

A content-addressed, immutable package store (Nix/OSTree lineage) with atomic,
transactional activation and instant rollback:

- Packages are hash-named and immutable; no in-place mutation.
- Activation: stage new generation → `secd` verifies signatures → atomic root
  swap; failure rolls back.
- Declarative: desired system state is a manifest that `pkgd` reconciles.
- A/B base image for safe OS upgrades.

Full detail: blueprint §12.
