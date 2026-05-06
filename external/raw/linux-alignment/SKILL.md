---
name: linux-alignment
description: Enforce reference-first Linux 2.6 alignment for Lite kernel work. Use when modifying Lite kernel code, renaming or moving symbols/files, changing subsystem behavior, changing user-visible ABI such as sysfs/procfs/devtmpfs/ioctl/device names, or claiming a change aligns Lite with Linux. Requires reading Linux reference code before editing, producing a mapping ledger, checking same-symbol same-file placement, and limiting each change to one explicit alignment difference.
---

# Linux Alignment

## Purpose

Use this skill to keep Lite kernel changes aligned with Linux 2.6 in symbol
names, file placement, public semantics, and lifecycle flow.

Allow subsetting Linux behavior. Do not replace Linux concepts with Lite-only
models.

## Required Triggers

Apply this skill before editing when the task includes any of:

- adding, deleting, renaming, or moving a function, structure, global, file, or
  directory;
- changing kernel-visible or user-visible behavior such as sysfs, procfs,
  devtmpfs, uevent, ioctl, device naming, mount behavior, or error returns;
- changing init/probe/remove, register/unregister, publish/unpublish,
  parent-child ownership, reference counts, object release, or teardown flow;
- describing a change as Linux, Linux 2.6, or kernel alignment.

For analysis-only tasks, use the evidence and ledger portions without editing.

## Core Rules

- Read Linux first, then Lite, then decide whether to edit.
- Same symbol, same semantic role, same corresponding file.
- If Linux has a direct concept, align to that concept instead of inventing a
  Lite-specific helper, manager, bridge, wrapper, or intermediate layer.
- If there is no direct Linux match, mark `NO_DIRECT_LINUX_MATCH` and include
  `Why`, `Impact`, and `Plan` before proceeding.
- If placement differs, prefer fixing placement before changing semantics.
- Make one alignment change at a time. Do not mix cleanup, optimization, and
  unrelated semantic fixes into the same patch.

## Pre-Edit Gates

### Gate 0: Mapping Ledger

Before editing, produce a ledger that covers affected:

- files;
- directories;
- functions;
- structs;
- globals/statics;
- public surfaces.

For each function and struct, include:

- `Linux symbol`: `linux2.6/<path>/<file>::<symbol>`;
- `Lite file`: current Lite location;
- `Placement`: `OK`, `DIFF`, or `NO_DIRECT_LINUX_MATCH`.

If `Placement=DIFF`, do not proceed to semantic edits unless the current stage
explicitly accepts the difference with `Why/Impact/Plan`.

### Gate 1: Reference Evidence

Before editing, complete all three:

1. Read the Lite current implementation.
2. Read the Linux 2.6 corresponding implementation.
3. State the single difference this step changes.

### Gate 2: One Difference

Each patch must target one concrete difference, for example:

- placement;
- naming;
- return code;
- lifecycle timing;
- ownership/refcounting;
- user-visible path or attribute shape.

### Gate 3: No Direct Match

If Linux has no direct match:

- do not add the concept by default;
- mark `NO_DIRECT_LINUX_MATCH`;
- write `Why/Impact/Plan`;
- ask the user if the change would create new architecture or policy.

## Checks

### Placement

Check whether every affected function, struct, and file lives under the Linux
2.6 corresponding path. Semantic similarity does not override placement.

### Naming

Check whether names are Linux terms. Flag new Lite-specific vocabulary unless it
is clearly private glue with no Linux-facing meaning.

### Semantics

Check user-visible and subsystem-visible behavior:

- paths and names;
- attribute fields;
- return codes;
- event format;
- ordering guarantees;
- blocking/synchronous behavior;
- failure behavior.

### Flow and Lifetime

Flag these as likely `Flow/Lifetime: DIFF`:

- init/probe/remove responsibilities are mixed;
- core registration and instance publication are mixed incorrectly;
- code binds objects by name lookup instead of owner relationships;
- derived objects outlive or detach from their owner unexpectedly;
- release paths skip references, inodes, devices, queues, or pending work;
- partial initialization rollback differs from Linux without explanation.

## Output Template

Use this report before edits, or summarize it in the final answer for small
changes.

```text
Linux Alignment Report

Change scope:
- files:
- directories:
- public surface:

Reference-first evidence:
- Linux file:
- Linux symbol:
- Lite file:
- This step only changes:

Mapping ledger:
- functions:
  - <symbol>: linux2.6/<path>/<file>::<symbol>, lite=<file>, placement=OK/DIFF/NO_DIRECT_LINUX_MATCH
- structs:
  - <symbol>: linux2.6/<path>/<file>::<symbol>, lite=<file>, placement=OK/DIFF/NO_DIRECT_LINUX_MATCH
- globals/statics:
- files:
- directories:
- NO_DIRECT_LINUX_MATCH:

Consistency:
- Naming: OK/DIFF -> ...
- Placement: OK/DIFF -> ...
- Semantics: OK/DIFF -> ...
- Flow/Lifetime: OK/DIFF -> ...

If DIFF:
- Why:
- Impact:
- Plan:
```

## Validation

For code edits, run the narrowest useful validation first. For Lite kernel
alignment stages, default validation is:

```sh
make -j4
make smoke-128
make smoke-512
```

If a validation step is not run, state why and name the exact command the user
should run.
