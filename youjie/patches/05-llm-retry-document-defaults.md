# llm-retry: state the billing effect of the default retry count and which adapters honour `Retry-After`

Patch: `05-llm-retry-document-defaults.patch` (base `dsh-v0.2.1-rc.2`, 687f282e8, same as `master` HEAD).

This is the documentation-and-tests variant of the "document or lower the default retry count" item. It deliberately does
not change the default.

## Problem

`dsh-llm-retry` defaults to normal mode with `DEFAULT_MAX_RETRIES = 5` (`packages/llm/llm/src/retry-policy.ts`): backoff of
500 ms doubling to a 10 s ceiling with 10 percent jitter. The package README already lists "five retries" and the eligible
error codes, but it never says what that means for the bill: a step whose upstream keeps failing sends **six** requests
(the first plus five retries). With a long prompt, and a provider that bills a request that fails late, that can be up to
six times the input cost of the step. The way to turn it off (`maxRetries: 0` in a normal-mode `retryPolicy`) is not stated
in the README either. The policy values are covered by unit tests of the policy itself, but no test drives a failing step
through the executor to pin the request count and the delays a user would observe.

A second surprise sits next to it. The executor honours a provider-supplied delay (`providerRetryAfterMs` on the adapter's
failure) before its own backoff. Only `dsh-llm-deepseek` fills it (from the `retry-after` response header,
`llm-deepseek/src/transport.ts`). `dsh-llm-pi-ai` cannot: pi-ai error events carry no response headers. Operators who
route through pi-ai therefore get the local 500 ms to 10 s schedule even when the provider asked for a longer pause, which is
easy to read as "the harness ignores Retry-After".

## Reproduction

Read the README, then ask: how many requests does one failing step send, and how do I make it one? Neither answer is
written down. With the new tests: a scripted adapter that fails every call receives 6 requests under the default policy
(`llm/retry` events carry `delayMs` 500, 1000, 2000, 4000, 8000 and `maxRetries: 5`), and exactly 1 request, with no
`llm/retry` event, under `maxRetries: 0`.

## Fix

- `packages/llm/llm-retry/README.md`, `README.zh.md` (pairing record re-recorded): in the "Minimal configuration"
  paragraph, "A failing step therefore sends up to six billed requests; set `maxRetries: 0` in a normal-mode
  `retryPolicy` to send each request once."; a new "Known limitations" bullet "Provider `Retry-After` depends on the
  adapter" with the deepseek/pi-ai split above.
- `packages/llm/llm-retry/tests/retry.spec.ts`: two end-to-end tests that pin the behaviour the new sentences describe.

No source change; the default stays at five.

## Risk

- None for runtime behaviour. The new tests would fail if someone changes `DEFAULT_MAX_RETRIES`, the first backoff steps,
  or the meaning of `maxRetries: 0`, which is intended: such a change should also update the README sentence.
- The bullet about adapters will go stale if `dsh-llm-pi-ai` (or another adapter) starts reporting `providerRetryAfterMs`;
  the fix is a one-line README edit and the executor already handles the field.

## Tests

`packages/llm/llm-retry/tests/retry.spec.ts`:

- `retries five times under the default normal policy, so a failing step sends six requests` (fake timers, fixed jitter,
  asserts the six requests, the five `delayMs` values, `maxRetries: 5` and the final `turn/end` error);
- `sends one request and records no retry when the normal budget is zero`.

## Verification performed

- `vitest run packages/llm/llm-retry`: 44 tests pass in the package folder (with coverage the package `src` stays fully
  covered).
- oxlint on the spec; `tsc -b packages/llm/llm-retry`: clean.
- Doc gates: `verify-translation-pairing`, `verify-md-wrap`, `verify-doc-budgets`, `verify-package-readme-limitations`,
  `verify-package-readme-summaries`, `verify-md-links`: pass.

## Alternatives considered

- Lower the default (for example to 2 or 3): changes behaviour for every deployment and trades resilience to transient
  provider errors for cost; that is a policy decision for the maintainers, so the patch only makes the current policy
  discoverable and pinned.
- Make the bullet adapter-agnostic by adding header support to pi-ai: requires an upstream pi-ai change; out of scope.
