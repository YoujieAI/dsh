# acp: offer the `_meta` of `session/new` and `session/resume` to plugins (optional)

Patch: `04-acp-session-meta-event.patch` (base `dsh-v0.2.1-rc.2`, 687f282e8, same as `master` HEAD).

This item is optional. It adds a small public event; maintainers may prefer a different shape or none at all.

## Problem

Every ACP request may carry a `_meta` object (an extension point of the protocol). An embedding client uses it to pass
per-session context that is not part of the standard parameters: a tenant or workspace id, a billing account, a trace id,
feature flags. `dsh-acp` validates the standard parameters and drops `_meta`, so a plugin running next to the bridge has
no way to learn what the client sent for a given session. The only workaround is to fork the bridge or wrap the JSON-RPC
stream.

The data must be available before the Agent is published: later hooks (`agent/created`, `session/created`, the MCP mounts)
run with the Agent already visible to other plugins, and a plugin that wants to bind the metadata to the session, or to
reject the request, has to act first.

## Reproduction

Send `session/new` with `_meta: { tenant: 'acme' }` to the bridge. No plugin event, service or hook receives
`{ tenant: 'acme' }` at any point; `session/new` succeeds as if `_meta` had not been sent.

## Fix

A new serial Cordis event, declared in `packages/acp/acp/src/index.ts`:

```ts
'acp/session-meta'(payload: {
  agent: Agent
  method: 'session/new' | 'session/resume'
  meta: Readonly<Record<string, unknown>>
}): undefined | Promise<undefined>   // @mode serial
```

- Dispatched with `ctx.serial` from the `setup` callback of `AcpSession.create` and `AcpSession.resume`, which runs after
  the Agent is composed and before it is published: before the model control and MCP servers mount and before
  `session/created`/`agent/created`.
- Dispatched only when the request has a `_meta` object (nothing for `undefined` or `null`).
- Listeners are awaited in order. A throw or a rejection fails the request and the unpublished Agent is rolled back by the
  existing setup-failure path, so a plugin can reject a session.
- The bridge keeps nothing; a listener that needs the metadata later stores it by `payload.agent.session.id`.
- `meta` is passed as received (unvalidated): the bridge makes no claim about its shape.

Documentation: the `session/new` and `session/resume` rows of the ACP README (en, zh, pairing record) mention the event;
the generated docs are regenerated because the event is public (`docs/subsystems/core*.md`, `docs/config-catalog*.md`,
`docs/event-producer-consumer*.md`, `packages/extensions/tool-cordis/src/api-catalog.ts`), and
`scripts/gen-cordis-catalog.ts` gets `'acp': 'core.md'` in `EVENT_SCOPE_PAGE`, because the generator refuses an event scope
with no page (`acp/*` had none).

## Risk

- New public event surface: the name, the payload shape and the "before publication" timing become a compatibility
  commitment. The payload is deliberately an inline object type so that no new exported type is added.
- No behaviour change for requests without `_meta`, and no listener means no work beyond one `ctx.serial` call.
- A listener that throws now fails `session/new`/`session/resume`. This is intentional (it is how a plugin rejects a
  session), but a buggy third-party listener can therefore break session creation.
- `meta` may contain anything the client sent, including large or hostile values; listeners must treat it as untrusted
  input. The bridge does not log it.
- The generated catalogs touched by the event (core subsystem page, config catalog, producer/consumer table, tool-cordis
  API catalog) grow by the event entry only.

## Tests

`packages/acp/acp/tests/bridge.spec.ts`:

- offers the `_meta` of `session/new` and `session/resume` to listeners while the Agent is unpublished (asserts the
  payload, the method, and that the Agent is not yet visible through `ctx.agents`);
- dispatches no meta event for a request without `_meta`;
- fails the request and publishes no Agent when a meta listener rejects.

## Verification performed

- `vitest run packages/acp/acp`: 153 tests pass (the native flock addon had to be built locally with
  `tsx native/system/scripts/build.ts --host-addon-only` because the install ran with `--ignore-scripts`; without it 110
  persistence tests fail on pristine and patched source alike).
- `tsc -b packages/acp/acp`, oxlint: clean.
- `gen-cordis-catalog` run to regenerate the files above; generated files committed in the patch.
- Doc gates: `verify-translation-pairing`, `verify-md-wrap`, `verify-md-links`, `verify-doc-budgets`,
  `verify-package-readme-limitations`: pass.

Not run: `verify-doc-site-fragments` (needs the docs site build) and the repository-wide gate lists.

## Alternatives considered

- Pass `_meta` into the Agent options or the Session header so it is persisted: bigger change, needs a session-format
  decision; the event lets a plugin choose where to persist.
- A service (`ctx.acpMeta`) that stores the latest `_meta` per session: the bridge would own state and its eviction.
- A hook on every ACP method: wider than the need (only session creation/resume binds a session to a client context).
