# Storage Documentation Index

This directory documents userspace storage management and administrative workflows for Wicofe.

## Scope
- Block-device discovery and presentation.
- Partition inspection and management.
- Mount orchestration and volume configuration.
- Filesystem formatting/checking frontends and storage diagnostics.
- Safe storage workflows and recovery handoff.

## Ownership
Kernel block I/O and device drivers belong to [Wicofe-Kernel](https://github.com/Waizusora/Wicofe-Kernel). ONWFS format and filesystem-specific tools belong to [ONWFS](https://github.com/Waizusora/ONWFS). This repository owns system-level storage workflows, not on-disk formats or kernel driver internals.

## Safety
Every destructive action must identify the exact target, validate capacity, display the planned changes, require explicit confirmation, and define interruption behavior. Device paths must not be assumed stable.

- [Central documentation index](https://github.com/Waizusora/Wicofe-Docs/blob/main/INDEX.md)
- [Repository documentation map](https://github.com/Waizusora/Wicofe-Docs/blob/main/REPOSITORY_DOCUMENTATION_MAP.md)
