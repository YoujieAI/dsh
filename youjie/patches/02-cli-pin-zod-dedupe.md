# apps/cli: declare the pinned zod so npm installs one copy, and make the install-layout gate see it

Patch: `02-cli-pin-zod-dedupe.patch` (base `dsh-v0.2.1-rc.2`, 687f282e8, same as `master` HEAD).

## Problem

66 workspace packages declare `"zod": "4.4.3"` (exact) and 7 declare `"^4.4.3"`. The published `@deepseek-ai/dsh` package
declares no zod of its own. Third-party dependencies in the closure accept a wider zod range, so when npm builds the tree
for `npm install @deepseek-ai/dsh` it hoists the newest zod that satisfies those ranges to `node_modules/zod`, and every
package with an exact `4.4.3` pin gets its own nested `node_modules/zod` copy.

Result for a plain `npm install` of `0.2.1-rc.2` today: one hoisted zod 4.6.7 plus 50 nested zod 4.4.3 copies. Besides
the disk and install-time cost, two zod copies in one process mean a schema built by one copy can be handed to code that
uses the other; that cross-copy interaction was not reproduced in this investigation and is only a reason to prefer one
copy, not part of the evidence.

The repository has a gate for exactly this kind of layout problem, `scripts/verify-npm-install-layout.ts`, but it cannot
see the problem: the registry it simulates serves only versions that appear in `pnpm-lock.yaml`, so npm never sees a zod
newer than 4.4.3 and the exact pins nest nothing.

## Reproduction

`npm install --package-lock-only --ignore-scripts @deepseek-ai/dsh@0.2.1-rc.2` against the public registry in an empty
directory, then count `zod` entries in `package-lock.json`. The public registry was reached read-only through a local
rewriting proxy, so the variants below differ only in the manifests the proxy rewrote:

| variant | zod copies in the lock |
| --- | --- |
| rc.2 manifests as published | 51 (50 nested 4.4.3, 1 hoisted 4.6.7) |
| every `^4.4.3` rewritten to exact `4.4.3` | 55 |
| every exact `4.4.3` rewritten to `^4.4.3` | 1 (4.6.7) |
| `@deepseek-ai/dsh` additionally declaring `zod: 4.4.3` (this patch) | 1 (4.4.3) |

In-repo gate before the patch, with a synthetic newer zod made visible to the gate's local registry: 52 zod installations,
verdict FAIL. After the patch: 1, verdict PASS (295 DSH packages per release, 2594 internal edges checked).

## Fix

- `apps/cli/package.json`: add `"zod": "4.4.3"` to `dependencies`, and update the `apps/cli` importer in `pnpm-lock.yaml`
  (specifier and resolved version only; no other lock change). With the root package declaring the exact version the
  packages pin, npm resolves the root edge first and hoists 4.4.3 above every pinning package, so no pinning package needs
  a nested copy. Third-party ranges that admit 4.4.3 also share it.
- `scripts/verify-npm-install-layout.ts`:
  - `withNewerRelease(index, name, version)` adds a release beside the locked ones, and `main()` uses it to add
    `NEWER_ZOD_VERSION = '4.999.0'`, so the gate finally sees a registry in which the exact pins can nest;
  - `assertDualDshInstallLayout` fails when more than one zod installation exists (message lists the paths and names the
    cause);
  - the success log line says that both releases share one Cordis and at most one zod installation.

The pin stays exact because 66 of the 73 packages that declare zod already pin it exactly; the patch does not change that
choice. An alternative is to relax the exact pins to `^4.4.3` (see below).

## Risk

- Users of `@deepseek-ai/dsh` get zod 4.4.3 instead of the newest 4.x in the shared slot. That is the version the code is
  tested against; the previous layout already ran 50 packages on 4.4.3.
- A third-party dependency requiring a zod range that excludes 4.4.3 still gets its own nested copy. The gate asserts "at
  most one", so such a dependency would be caught the next time it is added.
- The synthetic `4.999.0` release only exists in the gate's local registry; nothing is published.
- A future zod bump has to change the pin in `apps/cli` together with the packages. No script in `scripts/` compares the
  `apps/cli` zod pin with the packages' pins, so the new layout assertion is what catches a mismatch (it fails once a
  newer zod exists and any package still pins an older one).
- Peer dependants that declare `zod` as a peer are unaffected.

## Tests

`scripts/verify-npm-install-layout.spec.ts`, new `describe('zod installation')`:

- accepts one shared zod installation;
- rejects zod copies nested under DSH packages (the failure message names the paths);
- `withNewerRelease` adds the release beside the locked ones without mutating the original index.

## Verification performed

- Targeted vitest of the spec above: pass.
- Full dual-release layout check (`tsx scripts/verify-npm-install-layout.ts` equivalent run through the same functions,
  with the real lock): HEAD fails with 52, patched tree passes with 1 (numbers above).
- Real-registry `npm install --package-lock-only` variants (table above).
- oxlint on both scripts.
- `verify-package-dependencies`, `verify-dependency-catalog`, `verify-optional-dependency-imports`: pass with the new
  dependency.
- A full `pnpm install` of the workspace (isolated store) with the changed manifest and lock completed without error.

Not run: the aggregated `check`/CI gate lists and anything that needs the website or docs build.

## Alternatives considered

- Relax the 66 exact pins to `^4.4.3`: npm then installs one zod, but the version drifts with the registry and the
  repository's exact-pin policy for runtime dependencies is lost.
- `overrides` in the published manifest: npm honours `overrides` only from the root project, so it cannot help users who
  install `@deepseek-ai/dsh` as a dependency.
- `peerDependencies` on zod: pushes the choice to every user and breaks `npm install -g`.
