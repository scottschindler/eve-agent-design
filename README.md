# eve-agent-design

A Claude Code skill that helps you design and build useful agents with [eve](https://eve.dev), Vercel's filesystem-first framework for durable AI agents.

Where the official `vercel/eve` skill points your coding agent at eve's bundled API docs, this skill supplies the **design methodology**: it interviews you about what you want to build, maps your requirements to the right eve features, then drives an incremental build with security hardening and evals baked in — so you end up with a working, tested, production-ready agent instead of a demo.

## What it does

1. **Discovery** — asks what the agent should do, who talks to it, what it integrates with, what's sensitive, and where it runs.
2. **Feature mapping** — turns those answers into an `agent/` directory plan using decision tables covering tools vs skills vs connections vs subagents, channels, schedules, state, sandbox, and dynamic capabilities.
3. **Incremental build** — scaffolds with `eve init`, then adds one surface at a time, verifying each with `eve info` / `eve dev` before moving on. Always defers to the version-matched docs in `node_modules/eve/docs` for exact APIs.
4. **Security hardening** — route auth, HITL approvals on sensitive actions, sandbox network policy, harness audit, channel signature verification, data minimization, and cost limits, with a pre-production gate.
5. **Evals** — smoke, negative, approval-flow, and multi-tenant probe evals, `mockModel` fixtures, and `eve eval --strict` in CI.

## Install

```bash
npx skills add scottschindler/eve-agent-design   # if published to the skills registry
```

or manually:

```bash
git clone <this repo> ~/.claude/skills/eve-agent-design
```

## Files

- `SKILL.md` — the five-phase workflow
- `references/feature-map.md` — every eve surface and when to use it
- `references/security-checklist.md` — trust model and hardening checklist
- `references/testing-and-evals.md` — eval strategy and CI gating

Built against eve beta docs as of July 2026. eve moves fast — the skill instructs the agent to verify APIs against the bundled docs of the installed version.
