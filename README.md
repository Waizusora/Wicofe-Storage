# Wicofe-Storage

Userspace storage utilities and storage-management integration for Wicofe OS.

## Scope

- Block-device discovery and presentation.
- Partition table inspection and management.
- Filesystem formatting and checking frontends.
- Mount orchestration and volume configuration.
- Storage diagnostics and administrative tools.

## Boundaries

Low-level block-device drivers and kernel block interfaces belong in Wicofe-Kernel. VFS and mount-system primitives are kernel responsibilities. ONWFS owns its filesystem format and filesystem-specific tooling; this repository may provide system-level frontends.

## Safety

Destructive operations must require explicit target identification, validate device identity and size, provide clear warnings, and avoid implicit formatting. Never assume a device path remains stable across discovery operations.

## Status

Initial repository baseline; no complete storage manager is claimed.

## License

Original project code targets MPL-2.0; external utilities and libraries require license review.

## Documentation

See [docs/INDEX.md](docs/INDEX.md) for this repository's technical documentation. The [central repository documentation map](https://github.com/Waizusora/Wicofe-Docs/blob/main/REPOSITORY_DOCUMENTATION_MAP.md) identifies canonical ownership and cross-repository boundaries.

## Wicofe-wide principles

Wicofe is a Unix-like, POSIX-compatible, non-GNU alternative. Native components must use Wicofe interfaces and must not require Linux libc or GNU compatibility layers. Prefer permissive licenses; avoid GPL/AGPL, and consider LGPL only when no suitable permissive alternative exists. Reuse or fork upstream software when implementing it from scratch would be disproportionately difficult. See the [canonical project principles](https://github.com/Waizusora/Wicofe-Docs/blob/main/PROJECT_PRINCIPLES.md).
