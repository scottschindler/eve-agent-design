# eve feature map — what to use when

Design-time decision guide for every eve surface. Verify exact signatures against `node_modules/eve/docs/` before writing code.

## Project shape

```text
my-agent/
├── package.json          # root agent name comes from "name"
├── agent/
│   ├── agent.ts          # runtime config: model, reasoning, compaction, limits
│   ├── instructions.md   # required on root: always-on system prompt
│   ├── instrumentation.ts# optional OTel telemetry (root-only)
│   ├── tools/            # typed actions (defineTool)
│   ├── skills/           # on-demand procedures (markdown / defineSkill)
│   ├── connections/      # MCP + OpenAPI servers (root + subagents)
│   ├── channels/         # inbound surfaces (root-only)
│   ├── hooks/            # stream-event observers
│   ├── sandbox/          # sandbox override + workspace seed files
│   ├── schedules/        # cron jobs (root-only)
│   ├── subagents/        # declared specialist child agents
│   └── lib/              # shared authored code, import-only
└── evals/                # app-root sibling of agent/, not inside it
    └── evals.config.ts   # required, exactly one
```

Identity is path-derived — never write a `name`/`id` field. `agent/tools/get_weather.ts` → tool `get_weather` (snake_case ASCII required for tools). Subagent name = its directory. Debug discovery with `eve info`; artifacts under `.eve/`.

## agent.ts (`defineAgent`)

- `model`: gateway id string (`"anthropic/claude-sonnet-5"`, routes via Vercel AI Gateway) or a provider `LanguageModel` (`anthropic("claude-...")` from `@ai-sdk/anthropic`, direct, needs provider key). Omit the file entirely for the default model. Note id formats differ: gateway `anthropic/claude-opus-4.8` vs direct `claude-opus-4-8`.
- `reasoning`: `"none" | "minimal" | "low" | "medium" | "high" | "xhigh"` (provider-mapped).
- `compaction.thresholdPercent`: default 0.9; lower for long sessions.
- `limits`: `maxInputTokensPerSession`, `maxOutputTokensPerSession` (cost guard — set these for anything with spend exposure), `maxSubagentDepth` (default 3).
- `outputSchema`: structured output for task-mode runs (subagent/schedule/remote-job).
- Subagent `agent.ts` additionally **requires** `description` (the parent routes on it).

## Instructions vs skills vs tools — the core triage

| Question | Yes → |
|---|---|
| Must the model always know it? | `instructions.md` |
| Is it a procedure needed only for some requests? | skill (progressive disclosure via `load_skill`) |
| Does it need to *execute* typed code? | tool |
| Is it an external service that already speaks MCP/OpenAPI? | connection |
| Does it need a different persona or narrower tool surface? | subagent |

Keep instructions short: identity, scope, hard rules, when to ask vs act. Everything procedural that isn't always relevant goes into skills — a loaded skill only adds text to context, never new execution surface.

## Tools (`defineTool` from `eve/tools`)

- `description` (for the model), `inputSchema` (Zod/Standard Schema/JSON Schema — required; `z.object({})` for none), `execute(input, ctx)`, optional `outputSchema`, optional `toModelOutput` (shrink what the model sees; channels/hooks still get full output), optional `approval`.
- `ctx`: `session` (id, auth, turn, parent), `abortSignal`, `getSandbox()`, `getSkill(id)`, `getToken(auth)` / `requireAuth(auth)` for connection-style auth inside tools (map downstream 401 → `ctx.requireAuth` to re-challenge).
- Tools run in the **app runtime** with full `process.env` — not the sandbox. Output must be JSON-serializable. Never return secrets or unminimized sensitive data.
- **Durability contract**: completed steps replay recorded results; interrupted steps re-run. Non-idempotent side effects (charges, emails) need idempotency keys or an `approval` gate.

## Human-in-the-loop

- `approval:` on a tool/connection with helpers from `eve/tools/approval`: `never()` (default when omitted!), `once()` (first call per session), `always()`, or a policy fn receiving `{ session, toolName, toolInput, approvedTools }` returning `"user-approval" | "not-applicable" | "approved" | "denied"` (or `{ type, reason }`). Guard by `session.auth.current` (this turn's caller) vs `session.auth.initiator` (session creator). Common patterns: amount thresholds, cross-tenant denial, auto-allow for app-principal (schedule) turns by matching `authenticator: "app"`, `principalId: "eve:app"`, `principalType: "runtime"`.
- Built-in `ask_question` tool: model asks the user `{ prompt, options?, allowFreeform? }` mid-turn.
- Both park the run durably at `session.waiting` (seconds or days, survives restarts) and resume via `inputResponses` or matching follow-up text. Channels render approvals/questions natively (Slack buttons, etc.).

## Skills

- Flat markdown (`skills/forecast.md`), packaged dir (`skills/research/SKILL.md` + `references/`, `scripts/` — description frontmatter required), or `defineSkill` in TS for generated content.
- The `description` is the routing hint — write it as the triggering task.
- Scoped per agent; subagent skills invisible to root and vice versa. Skill files are seeded into `/workspace/skills/` in the sandbox; read siblings at runtime via `ctx.getSkill(id).file(path)`.

## Connections

- `defineMcpClientConnection({ url, description, auth?, headers?, approval? })` or `defineOpenAPIConnection({ spec, baseUrl?, ... })` — one per file under `connections/`. Model discovers tools via built-in `connection_search`, calls them as `<connection>__<tool>`. URLs and tokens never reach the model.
- **Auth decision**:
  - App-scoped (shared credential): `auth: { getToken }` reading env/secret manager; `expiresAt` for TTL refresh.
  - User-scoped (each end user's own account): `connect("uid")` from `@vercel/connect/eve` (Vercel Connect OAuth, encrypted storage, refresh), or self-hosted `defineInteractiveAuthorization` (getToken / startAuthorization / completeAuthorization). Requires route auth resolving `principalType: "user"` — otherwise fails with `principal_required`.
  - Per-caller/tenant: make `auth`/`headers` functions of `ctx`.
- `approval: once()/always()` puts every connection tool behind a human.

## Channels

Root-only, under `channels/`. Default eve HTTP channel is always on (session routes + stream); author `channels/eve.ts` mainly to set route auth. Scaffold with `eve channels add [slack|web]`.

| Surface | Channel |
|---|---|
| Web app / browser chat | eve channel + `useEveAgent` (React/Vue/Svelte guides in docs) |
| curl / SDK / service-to-service | eve channel (default) |
| Slack, Discord, Teams, Telegram, Twilio (SMS/voice), GitHub, Linear | built-in channels |
| Anything else (webhooks, WebSocket) | `defineChannel` custom channel |

Channels own inbound auth (see security checklist) and the `continuationToken`. Schedules deliver to channels via `receive`.

## Sandbox

Every agent has exactly one; built-in `bash`/`read_file`/`write_file`/`glob`/`grep` target it; authored code via `ctx.getSandbox()` (`run`, `spawn`, read/write files, `setNetworkPolicy`). `/workspace` persists across turns of a session.

Author `agent/sandbox/sandbox.ts` (`defineSandbox`) only for: seeding files (`sandbox/workspace/**`), setup (`bootstrap` = template-scoped, once; `onSession` = per-session, has `ctx`), backend choice, or network policy. Backends: `defaultBackend()` picks Vercel Sandbox on Vercel → Docker → microsandbox → just-bash locally. Network policy: `"allow-all"` (default) | `"deny-all"` | domain allow-list with per-domain credential-brokering `transform` (secrets injected at the firewall, never inside the sandbox). Docker backend only honors allow-all/deny-all; just-bash has no isolation or real binaries.

## Subagents

- **Built-in `agent` tool** (zero authoring): copy of the current agent, shares sandbox + tools, fresh history/state. Use for parallel fan-out of independent subtasks (batch multiple calls in one response). Give children non-overlapping write scopes.
- **Declared subagent** (`subagents/<id>/`): own instructions/tools/connections/skills/sandbox/hooks; inherits *nothing* from the root; `description` in its `agent.ts` is required. No channels or schedules inside. Tool name = bare directory name (collides with same-named tools → build error).
- Prefer a skill when the agent can keep its identity and just needs a procedure. Delegation is **not** a security boundary — sensitive tools need their own approval wherever callable.
- Remote agents: call another eve deployment as a subagent (see `guides/remote-agents`).

## Schedules

Root-only, `schedules/<name>.ts` or `.md`. `cron` (5-field, **UTC on Vercel**, minute granularity) + exactly one of:
- `markdown`: fire-and-forget task-mode prompt (cannot park for approval/OAuth — design accordingly; give the schedule's tools an app-principal approval skip if needed).
- `run({ receive, waitUntil, appAuth })`: handler that computes arguments and hands off to a channel via `receive` (e.g. post digest to Slack).

`eve dev` never fires cron; test via the dispatch route or `eve start`. On Vercel each becomes a Vercel Cron Job. For per-user/dynamic cadences see `patterns/dynamic-scheduling`.

## State, hooks, dynamic capabilities

- `defineState(name, initial)` from `eve/context`: typed durable per-session slot (`get()`/`update(fn)`), survives crashes/redeploys. Never shared with subagents. Cross-session or queryable data belongs in a real store/connection. Reset per turn via a `turn.started` hook if needed.
- `defineHook` (`hooks/`): observe stream events (`session.started`, `turn.completed`, `message.completed`, `action.result`, `*`). Observe-only — cannot inject context; a thrown hook fails the turn. Use for audit logs, metrics, mirroring sessions to your DB. `toolResultFrom(event.data.result, tool)` narrows typed tool outputs.
- `defineDynamic` (in `tools/`, `skills/`, or `instructions/`): resolve capabilities per session/turn from `ctx.session.auth` — per-tenant tool sets, per-team playbooks, feature flags. Dynamic tool `execute` **must be inline** (bundler reconstructs closures across replays). Resolver events: `session.started`, `turn.started`, (`step.started` tools-only).

## Default harness (built-in tools)

`bash`, `read_file`, `write_file`, `glob`, `grep` (proxy into sandbox), `web_fetch`, `web_search`, `todo`, `ask_question`, `agent`, `load_skill` (when skills exist), `connection_search` (when connections exist). Override by authoring a tool at the same slug (spread the default from `eve/tools/defaults` to keep its wiring); disable with `disableTool()` sentinel. Audit these in Phase 4 — e.g. disable `bash`/`web_fetch` for agents that shouldn't shell out or fetch arbitrary URLs.

## CLI quick reference

| Command | Use |
|---|---|
| `npx eve@latest init <name>` / `eve init .` | scaffold new / add to existing |
| `eve info [--json]` | first debugging move: discovered surface + diagnostics |
| `eve dev [--no-ui]` / `eve dev <url>` | local dev server / drive a remote deployment |
| `eve build` | compile `.eve/` artifacts + host output |
| `eve start` | serve built output (self-host / prod schedules locally) |
| `eve eval [pattern] [--url] [--strict]` | run evals |
| `eve link` / `eve deploy` | link Vercel project / deploy production |
| `eve channels add [slack\|web]` | scaffold a channel |
