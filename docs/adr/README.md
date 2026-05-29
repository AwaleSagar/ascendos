# Architecture Decision Records

An ADR captures one architectural decision: its context, what we decided, and —
most importantly — the consequences and the alternatives we rejected. The
rejected options are the point. A decision without its discarded alternatives is
just an assertion.

We use a lightweight [Nygard-style](https://github.com/architecture-decision-record/architecture-decision-record)
format. New ADRs get the next number and are never edited to reverse a decision —
instead, a later ADR supersedes an earlier one, so the history survives.

## Index

| # | Title | Status |
|---|---|---|
| [0001](0001-pure-capability-model.md) | Pure capability model over ACLs/ambient authority | Accepted |
| [0002](0002-userspace-drivers.md) | Device drivers run in user space | Accepted |
| [0003](0003-ring-based-syscall-interface.md) | Ring-based syscall interface | Accepted (contested) |
| [0004](0004-no-native-kernel-modules.md) | No native loadable kernel modules | Accepted |

New decisions start from [`0000-template.md`](0000-template.md).
