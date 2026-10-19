# app-boot: report a profile patch whose target row does not exist

Patch: `03-warn-unmatched-patch-row.patch` (base `dsh-v0.2.1-rc.2`, 687f282e8, same as `master` HEAD).

## Problem

A launcher profile can carry patches (`{ id: 'tool-gaol', disabled: true }`, `{ id, insert: [...] }`, name-guarded
patches). When a patch refers to a row id that the composed profile does not contain (a typo, a row renamed in a release,
a row removed by a newer composition), the patch is silently dropped and the plugin it was meant to disable stays enabled.
For a `disabled: true` patch that is a security-relevant silent failure: the operator believes a plugin is off.

The app-boot README promises it ("A patch naming an entry that does not exist prints a stderr warning"), and
`applyEntryPatches` does call a warning callback for them. The callback in `prepareProfilePatches` is `patchWarning(ctx)`, which calls `ctx.logger.warn`. But:

- `prepareProfilePatches` flattens the patches into the composed rows before the `Include` entry sees them, so nothing
  downstream can report them later;
- in the `dsh` profiles the cordis logger has only the in-memory buffer; there is no console exporter. `boot()` installs a
  collector exporter whose output is only used for `StartupError`.

So the warning is written to a logger nobody reads, and the boot is silent.

## Reproduction

1. Boot a profile whose composition resolves to rows `row-1 ...`, with the patch `{ id: 'row-9', disabled: true }`.
2. Observe stderr and stdout: nothing. (`ctx.logger.warn` was called, but the buffer is not exported.)

The new boot-level test does this and expects `preflight: patch: entry "row-9" not found` on stderr.

## Fix

`packages/boot/app-boot/src/compatibility-preflight.ts`:

- `patchWarning` takes a `report(line)` callback instead of a context.
- `prepareProfilePatches` reports each unmatched patch with `process.stderr.write(`${binName}: ${line}\n`)`, the same
  channel and prefix that the existing "denied plugin" reports of this module use. A comment records why stderr is used.
- The Include-file preflight (`includedConflicts`) is behaviourally unchanged: it still reports through `ctx.logger.warn`.
  Whether that path deserves the same treatment is left to the maintainers; this patch only fixes the path that the README
  documents and that profile patches take.

The wording is unchanged (it is the text `applyEntryPatches` already produces, with `%C` replaced by the JSON of the
argument): `patch: entry "row-9" not found`, `patch: name mismatch for "row" (expected ..., got ...), skipping`,
`patch insert: entry "x" not found`, `patch insert: entry "x" is not a group`, `patch: id is required for non-insert patches`.

## Risk

- New stderr lines appear for profiles that already carried dead patches. Anyone parsing stderr of `dsh` start-up sees them
  (one line per dead patch, prefixed with the launcher name). That is the intent; it does not change the exit code.
- Patches that match are not reported; nothing else is written. The function still returns the same rows.
- Writing to `process.stderr` directly is consistent with the neighbouring reports; routing it through the logger would
  require a console exporter in the profile tree, which is a larger design change.

## Tests

`packages/boot/app-boot/tests/compatibility-preflight.spec.ts`:

- the existing "unmatched patch" unit test now expects the stderr line and that the logger is not called;
- `names the launcher and the patch kind in each unmatched-patch report` covers all five message shapes and a custom
  launcher name;
- `reports a patch for an absent row on stderr when the booted tree has no logger exporter` is the boot-level repro.

## Verification performed

- `vitest run packages/boot/app-boot` with coverage for the changed file: pass, no uncovered lines in `src`.
- `tsc -b packages/boot/app-boot`, oxlint: clean.

## Alternatives considered

- Throw on an unmatched patch: stricter, but breaks profiles that deliberately carry patches for rows that only exist in
  some compositions.
- Add a console exporter to the logger of the profile tree: broader behaviour change (it would also surface other, noisy
  warnings).
