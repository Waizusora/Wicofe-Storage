# Wicofe Userspace Storage Management Design

- **Version:** 0.1
- **Status:** Draft architecture
- **Repository:** Wicofe-Storage
- **Targets:** x86-64 and AArch64
- **License:** MPL-2.0 for original project code

## 1. Purpose and authority

Wicofe-Storage provides userspace storage discovery, inspection, administration, mount orchestration, and diagnostics. It is not the kernel block layer, VFS, filesystem implementation, or ONWFS format authority.

Canonical ownership:
- **Wicofe-Kernel:** block-device drivers, block I/O contract, VFS, mount primitives, permissions, and kernel filesystem integration.
- **ONWFS:** ONWFS on-disk format, formatter, checker, inspector, recovery, and format-specific implementation.
- **Wicofe-Storage:** system-level workflows, device presentation, partition-management frontends, mount orchestration, diagnostics, and administrative policy.
- **Wicofe-Services:** privileged service lifecycle and service IPC policy where applicable.
- **Wicofe-ABI:** public syscall and userspace ABI contracts.

This repository must not duplicate ONWFS serialization rules or implement an independent formatter that can drift from the canonical ONWFS specification and tools.

## 2. Design principles

1. **Explicit target identity:** destructive operations bind to a verified device identity, not only a transient path such as `/dev/sdX`.
2. **No implicit destruction:** discovery, inspection, and mount operations must never format or repartition as a side effect.
3. **Preview before mutation:** show the exact device, partition table, filesystem, offsets, sizes, and planned changes before confirmation.
4. **Least privilege:** separate read-only inspection from privileged mutation.
5. **Fail closed:** ambiguous identity, stale discovery data, unsupported layouts, and uncertain I/O state block destructive operations.
6. **Recoverability:** define interruption and post-failure behavior for every mutating workflow.
7. **Format ownership:** delegate filesystem-specific operations to the filesystem's canonical tools.

## 3. Logical components

The initial design is divided into these components; this is a logical decomposition, not a claim that executables already exist.

### 3.1 Discovery and inventory

Collects block-device information from documented kernel interfaces:
- Stable device identity and current kernel handle/path.
- Capacity and logical/physical sector sizes where available.
- Read-only/removable state.
- Bus and transport information.
- Device model/serial identifiers where available and permitted.
- Supported flush, discard, and other relevant capabilities.
- Current partition and mount associations.

The inventory must distinguish unknown values from zero or empty values. Sensitive identifiers should not be exposed to unprivileged users without a policy reason.

### 3.2 Partition inspection

Reads partition tables without modifying them. The initial design should support safe recognition of common partitioning schemes, with GPT and MBR candidates subject to implementation planning.

The parser must:
- Validate signatures and header sizes.
- Check all offsets, lengths, and arithmetic for overflow.
- Ensure partitions remain within device capacity.
- Detect overlaps and invalid entry ranges.
- Handle protective/hybrid MBR cases explicitly.
- Report unknown partition types without guessing their contents.
- Preserve unknown metadata during read-only inspection.

Partition table writes require a separate transaction and recovery design. Do not assume a multi-sector update is atomic.

### 3.3 Filesystem identification

Filesystem probing is read-only and confidence-based. A signature match is not proof that the filesystem is healthy or safe to mount read-write.

The probe result should include:
- Candidate filesystem type.
- Confidence and evidence (magic/version/features).
- Recognized and unknown feature flags.
- Whether the filesystem is known to support read-only or read-write mounting.
- Whether a checker or recovery action is recommended.
- Any ambiguity or conflicting signatures.

Never automatically run repair because a probe reports corruption.

### 3.4 Mount orchestration

Coordinates user intent with the kernel's mount interface:
- Resolve the selected volume to a current verified device identity.
- Validate requested filesystem and mount options against reported capabilities.
- Apply policy for removable, untrusted, or system-critical volumes.
- Request mount/unmount through documented privileged interfaces.
- Track mount state and report busy, stale, or detached devices.
- Never claim successful unmount until the kernel confirms it.

Mount options must be typed and validated, not passed as unchecked strings into privileged parsers.

### 3.5 Administration and diagnostics

Provides controlled workflows for:
- Listing devices, partitions, filesystems, and mounts.
- Inspecting capacity and filesystem health information.
- Formatting through the canonical formatter.
- Checking through the canonical filesystem checker.
- Mounting and unmounting.
- Ejecting or safely removing removable devices where supported.
- Collecting diagnostics without modifying the target.

A command-line interface and graphical frontend may share a common policy/service layer. UI code must not directly bypass safety checks.

## 4. Stable device identity and race prevention

A path is a locator, not a durable identity. Device enumeration can change between preview and execution.

Before a destructive operation:
1. Resolve the selected path to a kernel-issued device identity or generation-bearing handle.
2. Record capacity, logical sector size, partition-table identity, and relevant device attributes.
3. Display the exact target and proposed changes.
4. Obtain explicit user confirmation.
5. Revalidate the same identity and generation immediately before the first write.
6. Abort if the device was removed, replaced, resized, or otherwise changed.
7. Keep the verified handle for the operation; do not re-resolve a mutable path between writes.

Serial numbers and model strings are supporting evidence, not sufficient identity on their own. If the kernel does not yet provide a stable identity/generation contract, destructive operations must remain disabled or require a separately reviewed safe mechanism.

## 5. Destructive-operation transaction model

Formatting, repartitioning, and repair can destroy data. Each operation must expose a plan and explicit commit boundary.

The workflow is:

1. **Discover:** enumerate candidate devices and current state.
2. **Inspect:** read partition and filesystem metadata.
3. **Plan:** compute exact byte/sector ranges and intended metadata changes.
4. **Preview:** show target identity, capacity, affected ranges, and irreversible effects.
5. **Confirm:** require an explicit confirmation tied to the displayed plan.
6. **Revalidate:** verify identity, generation, size, and preconditions again.
7. **Execute:** perform the canonical operation with bounded progress and error reporting.
8. **Verify:** reread relevant metadata and validate the resulting state.
9. **Report:** distinguish complete, failed-before-write, partially-modified, and indeterminate outcomes.

A UI confirmation is not a substitute for service-side validation. The privileged backend must independently enforce all preconditions.

### 5.1 Interruption behavior

Every mutating operation must document:
- Which writes may have occurred at each stage.
- Whether the operation can resume safely.
- Whether rollback is possible or impossible.
- How partial state is detected.
- Which recovery tool is authoritative.
- Whether the device must remain offline or read-only.

Do not offer a generic "undo" when the underlying operation cannot guarantee rollback.

## 6. Partition management

Partition table editing is not initially assumed safe or implemented. Before enabling writes, the design must specify:
- Supported partition table formats and versions.
- Sector-size handling and alignment rules.
- GPT primary/backup header and entry-array update ordering.
- CRC validation and recomputation.
- MBR primary/extended/logical partition constraints.
- Hybrid MBR policy.
- Partition overlap and reserved-region validation.
- Backup and restore strategy.
- Crash/interruption recovery behavior.
- Kernel notification/rescan behavior after changes.

A partition editor must refuse unknown or ambiguous layouts rather than rewriting metadata it does not understand.

## 7. Filesystem tools integration

Filesystem-specific operations are delegated to canonical implementations:
- ONWFS format/check/inspect/recovery commands belong to the ONWFS repository.
- Other filesystem tools must be selected and packaged with license/provenance review.
- The storage layer may provide a common invocation interface, progress model, and result normalization, but must not reimplement each filesystem's on-disk metadata.

The integration contract should pass an already verified device handle or a securely revalidated device identity, explicit operation mode, and structured options. It must not rely on shell-string concatenation. Child processes should receive structured arguments and controlled file descriptors.

A filesystem checker must distinguish:
- Clean.
- Errors found but not modified.
- Repaired.
- Unrepairable.
- Interrupted.
- Unknown/unsupported feature.
- I/O failure.

## 8. Privilege and IPC model

Read-only inventory and inspection should be available with the least privilege compatible with kernel policy. Mutating operations require an authorized privileged service or equivalent kernel-mediated authorization.

The service interface must:
- Authenticate the caller through the OS security model.
- Authorize each operation independently.
- Validate all fields and bounds.
- Enforce confirmation tokens or plan identifiers server-side.
- Reject stale plans and replayed mutation requests.
- Return structured status and diagnostics.
- Avoid exposing raw privileged device access to UI processes.

Exact service IPC and authorization mechanisms are owned by the relevant Wicofe service and ABI specifications and remain open here.

## 9. Error model

Use structured errors that distinguish:
- Device not found or identity changed.
- Unsupported device or partition layout.
- Invalid user-supplied parameters.
- Permission denied.
- Busy or mounted target.
- Read-only media.
- Insufficient capacity.
- Filesystem unsupported or feature mismatch.
- I/O error or device removal.
- Operation interrupted.
- Partial modification.
- Indeterminate completion.

An error after writes begin must not be reported as if no changes occurred. Include the stage reached and whether the target may have been modified.

## 10. Logging and privacy

Administrative logs should record operation type, stable operation ID, target identity in a privacy-aware form, plan summary, authorization result, start/end status, and failure stage.

Do not log file contents, encryption keys, credentials, or unnecessary serial numbers. Logs must not be treated as proof that storage writes reached persistent media unless the kernel/device durability contract confirms it.

## 11. Testing strategy

### Discovery and parsing
- Empty device inventory and hotplug churn.
- Duplicate or incomplete identifiers.
- Device removal during enumeration.
- Malformed GPT/MBR headers and entry arrays.
- Integer overflow, out-of-range offsets, overlaps, and truncated media.
- Unknown partition types and filesystem signatures.

### Identity and safety
- Device path reused for a different device.
- Capacity or generation changes between preview and commit.
- Device removal immediately before and during mutation.
- Replayed or stale confirmation token.
- Unauthorized caller and privilege escalation attempts.
- UI/backend plan mismatch.

### Mutating workflows
- Failure before first write.
- Failure after each write boundary.
- Flush failure and device timeout.
- Power interruption and partial partition-table updates.
- Formatter/checker exit failures and malformed progress output.
- Verification mismatch after apparent success.

### Mount lifecycle
- Busy mount and unmount.
- Device removal while mounted.
- Unsupported mount option.
- Read-only fallback and unknown filesystem feature.
- Concurrent mount requests for the same volume.

All destructive tests must use disposable disk images or dedicated test devices. Never use a user's production storage as a test fixture.

## 12. Suggested repository layout

```text
src/
  discovery/
  partition/
  filesystem_probe/
  mount/
  operations/
  policy/
  diagnostics/
docs/
  INDEX.md
  STORAGE_MANAGEMENT_DESIGN_0.1.md
  PARTITION_TABLE_WRITE_PROTOCOL.md  # when separately reviewed
tests/
  images/
  discovery/
  partition/
  safety/
  integration/
```

This is a target organization, not a statement that these directories or components already exist.

## 13. Decisions still open

- Kernel stable block-device identity and generation interface.
- Exact userspace block-device enumeration API.
- GPT/MBR initial read-only scope and write support schedule.
- Privileged storage service and IPC contract.
- Mount syscall/service interface and option schema.
- Canonical ONWFS tool command/IPC contract.
- Operation plan/confirmation token design.
- Logging and privacy policy.
- Initial CLI/UI split and executable names.

## 14. Completion criteria

The storage subsystem is ready for destructive production use only after stable device identity is available, partition writes have a reviewed interruption protocol, privileged authorization is enforced server-side, filesystem tools are invoked through canonical interfaces, and fault-injection tests demonstrate safe behavior.

**This document is a draft architecture. It does not claim a storage manager, partition editor, formatter, or mount service is implemented.**
