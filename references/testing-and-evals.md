# Testing an eve agent

eve's native test surface is **evals**: scored checks that drive the real agent over its real HTTP surface and grade what came back. A passing eval means the agent booted, accepted a request, ran the loop, and produced the asserted behavior. Unit-test pure logic in `lib/` with your normal test runner; test *agent behavior* with evals.

## Layout

```text
evals/
├── evals.config.ts          # required, exactly one — shared defaults
├── smoke.eval.ts            # id: smoke
├── data/                    # fixtures
└── refunds/
    └── requires-approval.eval.ts   # id: refunds/requires-approval
```

`*.eval.ts` files under app-root `evals/` (sibling of `agent/`, not inside it). Path = identity. A file default-exports one `defineEval` or an array (dataset fan-out).

```ts
// evals/evals.config.ts
import { defineEvalConfig } from "eve/evals";
export default defineEvalConfig({
  judge: { model: "openai/gpt-5.4-mini" }, // omit if fully deterministic
});
```

## Anatomy

```ts
import { defineEval } from "eve/evals";
import { includes } from "eve/evals/expect";

export default defineEval({
  description: "Weather question uses the tool and reports conditions.",
  async test(t) {
    await t.send("What is the weather in Brooklyn?");
    t.succeeded();                                // gate: run completed
    t.calledTool("get_weather");                  // gate: right tool ran
    t.check(t.reply, includes("Sunny"));          // gate: content
  },
});
```

Drive: `t.send`, `t.respond`/`t.respondAll` (answer approvals/questions), `t.sendFile`, `t.requireInputRequest`, `t.newSession`. Read: `t.reply`, `t.events`, `t.sessionId`.

Assert on three surfaces:
- Run-level methods: `t.succeeded()`, `t.calledTool(name)` — gates by default.
- `t.check(value, assertion)` with `includes`/`equals`/`matches` (gates) or `similarity` (soft).
- `t.judge.autoevals.*` — LLM-as-judge, soft by default, uses the config judge model (never the agent under test).

Severity is chainable: `.gate(threshold?)` promotes, `.soft()` demotes to tracked metric, `.atLeast(0.7)` sets a soft bar that `--strict` enforces. `await t.require(...)` for a must-pass-before-continuing gate; `t.skip(reason)` for unsupported targets.

## The recommended suite

1. **Smoke per core job** — happy path: succeeded + right tool + content check.
2. **Negative behavior** — the agent does *not* act when it shouldn't:
   ```ts
   await t.send("hey there");
   t.succeeded();
   t.check(t.toolCallCount?.() ?? 0, equals(0)); // or assert specific tools weren't called
   ```
   (Confirm the exact no-tool assertion in the bundled docs — `evals/assertions.mdx`.)
3. **Approval flow** — for every `approval`-gated tool, prove the pause:
   ```ts
   await t.send("Refund charge ch_123 for $2000");
   const req = await t.requireInputRequest();  // parked at the approval
   await t.respond(req, "approve");
   t.succeeded();
   t.calledTool("refund_charge");
   ```
   And a denial variant asserting the side effect did not happen.
4. **Multi-tenant probe** (if applicable) — a session authenticated as tenant A asking for tenant B's data is refused/denied.
5. **Multi-turn / memory** — scripted `t.send` sequences asserting state carries across turns.
6. **Judge assertions** — only for fuzzy quality (tone, citation quality): `t.judge.autoevals.factuality(reference).atLeast(0.7)`.

## Deterministic runtime tests with mockModel

To exercise eve's runtime (tools, approvals, harness wiring) without provider calls or flakiness, give a fixture agent a scripted model:

```ts
// agent/agent.ts of a dedicated fixture agent
import { defineAgent } from "eve";
import { mockModel } from "eve/evals";

export default defineAgent({
  model: mockModel(({ toolResults }) =>
    toolResults.length === 0
      ? { toolCalls: [{ name: "get_weather", input: { city: "Brooklyn" } }] }
      : `Weather: ${JSON.stringify(toolResults[0]?.output)}`,
  ),
});
```

Use mockModel fixtures for plumbing correctness; use the real model for behavioral/routing evals. Mock external services inside tools the way you would in any integration test (env-switched base URLs, test-mode API keys).

## Running and CI

```bash
eve eval                     # all evals against a local dev server it boots
eve eval refunds             # one eval or a directory subtree
eve eval --url https://…     # target a deployed preview/prod
eve eval --strict            # soft threshold misses also fail — use in CI
```

Exit 0 = all gates passed. In CI: build, then `eve eval --strict`; optionally run the same suite against the preview URL post-deploy (`--url`). Add a Braintrust reporter in `evals.config.ts` when the team needs shared result review; JUnit XML reporter exists for CI dashboards.

## Also verify (outside evals)

- `eve info` clean (no discovery diagnostics) — cheap first CI step.
- `eve build` succeeds (a failed sandbox prewarm fails hosted builds by design).
- Schedules: `eve dev` never fires cron — trigger via the dispatch route in dev, or `eve start` for production-like behavior, and keep schedule logic thin enough that the prompt/handler is covered by an eval-style task run.
- TypeScript: the scaffold includes `evals/**/*.ts` in `tsconfig.json`; keep `tsc --noEmit` in CI so tool schemas and eval code type-check.
