# llm-pi-ai: keep the model's text when pi-ai could not parse a tool call's arguments

Patch: `01-pi-ai-keep-unparsable-tool-args.patch` (base `dsh-v0.2.1-rc.2`, 687f282e8, same as `master` HEAD).

## Problem

`dsh-llm-pi-ai` converts the pi-ai event stream into harness `StreamChunk`s. On `toolcall_end` it emits
`arguments: JSON.stringify(event.toolCall.arguments)`.

Since pi-ai 1.3.0 a finished `ToolCall` can carry `argumentsError`: when the model's complete argument text is not a JSON
object (truncated output, trailing comma, single quotes, a bare string, an array), pi-ai leaves `arguments` as `{}` and
sets `argumentsError` to a description that includes the text it received. The adapter ignores that field, so the harness
sees the model's malformed call as a perfectly valid empty-object call:

- a tool whose parameters are all optional runs with no arguments (the model asked for something else);
- a tool with required parameters fails with a misleading "missing property" message instead of "your arguments were not
  valid JSON", so the model cannot tell what it did wrong and tends to repeat the mistake;
- the original text is lost from the transcript.

The rest of the pipeline already handles the right case. `ToolCallBlock.arguments` is documented as the "raw JSON string
as produced by the model", the agent loop's `parseArguments` keeps unparsable text as a string, and dsh-tools
`validateArgs` reports `"arguments" must be an object` for a non-object. Only the pi-ai adapter hides the failure before
those layers see it.

The package range is `^1.1.0`, so npm consumers resolve a pi-ai that has the field.

## Reproduction

Real pi-ai 1.3.0, all three wire APIs (`completions`, `anthropic`, `responses`), five malformed argument texts each
(15 cases), through `toStreamChunks`:

| text sent by the model | `arguments` before this patch | after |
| --- | --- | --- |
| `{"title":"a","body":` (truncated) | `{}` | the same text |
| `{"title":"a",}` | `{}` | the same text |
| `{'title':'a'}` | `{}` | the same text |
| `["a"]` | `{}` | the same text |
| `hello world` | `{}` | the same text |

After the patch every one of the 15 cases is rejected downstream with `"arguments" must be an object`, instead of running
the tool with `{}`.

Unit-level repro: feed `toStreamChunks` a `toolcall_end` event whose `toolCall` is
`{ type: 'toolCall', id, name, arguments: {}, argumentsError: 'Failed to parse ...' }` after a `toolcall_delta` of
`{"a":`; the emitted block carries `arguments: '{}'` before the patch.

## Fix

`packages/llm/llm-pi-ai/src/stream.ts`:

- `toStreamChunks` remembers the concatenated `toolcall_delta` text per content index (the existing `toolIds` map gains a
  `streamed` field).
- A new helper `toolCallArguments(toolCall, streamed)` returns `JSON.stringify(toolCall.arguments)` when there is no
  non-empty `argumentsError`, otherwise the streamed text, or the `argumentsError` description (which embeds the received
  text) when nothing non-blank was streamed.
- The field is read with `'argumentsError' in toolCall && typeof ... === 'string'` because it is absent from the types of
  pi-ai releases before 1.3.0, which the dependency range still admits. No new dependency, no type cast.

README (en, zh, pairing record): the model-experience paragraph and the "streamed JSON tool-call arguments" limitation
state the new behaviour.

## Risk

- Behaviour change is limited to calls pi-ai itself flagged as unparsable. A call that parses is byte-identical to before.
- A tool that previously "succeeded" on `{}` after a malformed call now returns a validation error to the model. That is
  the intended outcome, and the agent loop already treats the error as a normal tool result so the model can retry.
- Providers that stream arguments only at `toolcall_end` (no deltas) fall back to the `argumentsError` text. The
  description is longer than the raw text, but it is still not an object, so validation rejects it the same way.
- **The monorepo's own lock pins pi-ai 1.1.0** (patched by `patches/@earendil-works__pi-ai@1.1.0.patch`, which removes the
  per-delta reparse). 1.1.0 has no `argumentsError`, so inside this repository the change is a no-op until pi-ai is bumped
  to 1.3.0 or later. The patch was therefore verified against a real 1.3.0 install (see below). A reviewer may prefer to
  land the bump first (or in the same PR) and re-base the local pi-ai patch; the bump is not part of this patch because
  it changes the lock and the patch file.

## Tests

`packages/llm/llm-pi-ai/tests/convert.spec.ts`, new `describe('tool call whose argument text pi-ai could not parse')`:

- keeps the streamed text instead of the empty arguments object;
- falls back to the pi-ai failure description when no argument text was streamed;
- treats an empty failure description as a parsed call (guards the `!failure` branch, keeps the 100% per-file coverage).

The tests build synthetic pi-ai events, so they pass on the lock's 1.1.0 as well.

## Verification performed

- `vitest run packages/llm/llm-pi-ai`: pass, with the coverage gate for `stream.ts` unchanged.
- oxlint and `tsc -b` of the package: clean.
- Real pi-ai 1.3.0 temporarily linked in place of 1.1.0: the 15-case matrix above, before and after.
- Doc gates: `verify-translation-pairing`, `verify-md-wrap`, `verify-doc-budgets`, `verify-package-readme-limitations`,
  `verify-md-links`: pass.

## Alternatives considered

- Re-parse the delta text in the adapter instead of reading `argumentsError`: duplicates pi-ai's parser and drifts from it.
- Emit a stream error for the whole turn: too coarse; one bad call among several valid ones would drop the valid ones, and
  the harness already has a per-call rejection path.
