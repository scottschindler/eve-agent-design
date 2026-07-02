# eve security checklist

Work through this before an agent touches real data or traffic. eve's defaults are permissive in two places (tool approval and sandbox egress) and strict in one (route auth fails closed) — know which is which.

## The trust model (hold this while designing)

|  | App runtime (trusted) | Sandbox (untrusted/model-driven) |
|---|---|---|
| `process.env` / secrets | yes | **no** |
| Your Node code, tools, connections, state | yes | no |
| Network | unrestricted | governed by network policy |
| Filesystem | app's own | isolated `/workspace` |

The model only ever sees tool descriptors and tool results. Secrets stay in the app runtime; connection tokens are injected into outbound requests and never serialized into session state or shown to the model. Built-in `bash`/file tools live in the app runtime and *proxy* into the sandbox. Design consequence: privileged operations belong in tools/connections (trusted side), and anything the model does in the sandbox should be treated as untrusted code execution.

## 1. Route auth (inbound) — fails closed, but finish the job

- [ ] Replace scaffolded `placeholderAuth()` in `agent/channels/eve.ts` with a real ordered `auth` walk. Helpers from `eve/channels/auth`: `vercelOidc()`, `httpBasic()`, `oidc()`, `jwtHmac()`, `jwtEcdsa()`, `none()` (explicit anonymous), `localDev()`, or a custom `AuthFn` that validates your app's own sessions/API keys and returns a `SessionAuthContext`.
- [ ] Order matters: your own authenticator first, `localDev()` last. Never ship `localDev()` alone — it trusts the Host header.
- [ ] Self-hosting outside Vercel? Don't rely on `vercelOidc()`; bring your own verifier.
- [ ] Multi-tenant or per-user OAuth connections? Route auth must resolve `principalType: "user"` with a stable `principalId`, and put the tenant in `attributes` so tools/approvals can read `ctx.session.auth.current.attributes.tenantId`.
- [ ] Verify after deploy: unauthenticated request to `POST /eve/v1/session` returns 401. (`GET /eve/v1/health` is intentionally public.)

## 2. Approvals (HITL) — the permissive default

Omitted `approval` = `never()`. Nothing pauses unless you say so.

- [ ] Enumerate every tool and connection tool that can: spend money, send messages/emails, create/modify/delete external data, access regulated or sensitive data, or take irreversible action. Each one gets `approval` — `always()`, `once()`, or a policy.
- [ ] Input-dependent policies: threshold on amounts, deny cross-tenant (`session.auth.current.attributes.tenantId !== toolInput.tenantId` → `{ type: "denied" }`), auto-allow only verified app-principal schedule turns (`authenticator === "app" && principalId === "eve:app" && principalType === "runtime"`).
- [ ] Non-idempotent side effects (charge, email): interrupted workflow steps re-run. Either gate with `always()` or use idempotency keys.
- [ ] Whole-connection gating: `approval: once()` on the connection puts every remote tool behind a human.
- [ ] Subagent delegation is NOT an approval boundary — gate the tool everywhere it's callable.

## 3. Sandbox — the other permissive default

- [ ] Default egress is `allow-all`. For anything handling non-public data set `networkPolicy: "deny-all"` or an explicit allow-list, in the backend factory or `onSession`'s `use()` (bootstrap-only config does not persist to sessions).
- [ ] Need authenticated egress from the sandbox (private git clone, authed curl)? Use credential brokering (per-domain `transform` injecting headers at the firewall) — never write secrets into `/workspace` or env inside the sandbox. Brokering + domain allow-lists work on `vercel()` and `microsandbox()` backends only; Docker is allow-all/deny-all; just-bash has no isolation at all — never rely on just-bash for security.
- [ ] Common pattern: factory open so `bootstrap` can install/clone, then lock down in `onSession`.

## 4. Harness audit

- [ ] Review built-ins: `bash`, `read_file`, `write_file`, `glob`, `grep`, `web_fetch`, `web_search`, `todo`, `ask_question`, `agent`, `load_skill`, `connection_search`.
- [ ] `disableTool()` what the agent shouldn't have (e.g. `bash` and `web_fetch` on a pure API-integration agent). Filename picks the tool; typos fail the build.
- [ ] Or override at the same slug, spreading from `eve/tools/defaults`, to add logging/guards while keeping behavior.

## 5. Channels

- [ ] Every platform channel has its signing secret configured (Slack, GitHub, Telegram, Twilio verify HMAC over the raw body).
- [ ] Custom channels: verify signatures in **constant time** (never `===` on a signature); derive identity from the verified signature/token, **never** from a body-supplied `principalId` — body fields are attacker-controlled.
- [ ] Escape model/user-controlled strings before rendering into channel UI markup.
- [ ] Disclosure: where law requires, tell users they're talking to an AI — eve does not add this automatically; put it in instructions/channel responses.

## 6. Data minimization

- [ ] Tool outputs: return only what the model needs (`toModelOutput` to project down); redact PII/credentials before returning from `execute`.
- [ ] Subagents: the `message` you pass is all the child sees — don't include sensitive data unless the child's tools/telemetry path warrant it.
- [ ] Connection tokens: scope to least privilege; they reach hosts but never the model.
- [ ] Telemetry (`instrumentation.ts`) and eval reporters (Braintrust) are data flows too — confirm they're appropriate for the data in sessions.
- [ ] Secrets live in env/secret manager only — never in source, compiled artifacts, `agent.ts`, or the sandbox.

## 7. Cost and blast-radius limits

- [ ] `limits.maxInputTokensPerSession` / `maxOutputTokensPerSession` in `agent.ts` sized to the use case (defaults are very high: 40M input on root sessions).
- [ ] `limits.maxSubagentDepth` (default 3) — lower it if delegation isn't needed.
- [ ] Budget-style guards for expensive tools via `defineState` counters (throw when a per-session cap is hit).

## Pre-production gate (run every item)

1. Unauthenticated prod request → 401.
2. Every sensitive/irreversible action pauses for approval (test one end-to-end through a channel).
3. Sandbox egress policy is intentional, not `allow-all` by accident.
4. Built-in tool surface reviewed; unneeded ones disabled.
5. Channel signatures verified; no body-derived identity.
6. Secrets audit: none in repo, artifacts (`.eve/`, `.output/`), or sandbox.
7. Multi-tenant only: cross-tenant probe eval passes (tenant A cannot read/act on tenant B — see `docs/patterns/multi-tenant-*`).
8. `eve eval --strict` green in CI.
