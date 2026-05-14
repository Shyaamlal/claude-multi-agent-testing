# Multi-Agent Testing with Claude Code

A dual-mode testing framework where specialist AI agents share the work and a human orchestrates — built on Claude Code + Playwright MCP.

## Why

Most "AI testing" today is one model acting as a co-pilot to a single tester. That works, but it doesn't scale and it doesn't separate concerns.

This framework distributes the work across eight specialist agents — each bounded to one task — with a human orchestrator at every judgment point. Outputs from Discovery mode graduate into automated Playwright spec files in Regression mode, which run in CI without consuming any tokens.

## The two modes

| Mode | Tool | Tokens | When |
|---|---|---|---|
| **Discovery** | Playwright MCP through Claude Code | Yes | New features, first passes, UX review, ambiguous behaviour |
| **Regression** | Playwright CLI agents (Planner / Generator / Healer) | None after generation | After every deploy, in CI |

A flow graduates from Discovery to Regression when:
- It passed cleanly in at least one Discovery session
- Expected behaviour is clearly defined
- UI elements are stable enough to trust selectors

## The agents (Discovery mode)

| Agent | Role |
|---|---|
| **Architecture Scout** | Maps the live application — what pages exist for each role, what actions are available, where the seams are |
| **Requirements Analyst** | Reads existing docs and surfaces ambiguity through targeted questions before execution begins |
| **Test Planner** | Sets execution order based on what Scout and Analyst found; replans after each area checkpoint |
| **Test Designer** | Derives executable steps 2-3 cases ahead from what the live UI actually shows — not from memory |
| **Test Executor** | Drives Playwright MCP one case at a time; reports raw observations, doesn't interpret |
| **Triage Agent** | Classifies findings (real bug / by design / automation artifact / ambiguous) with a 2-attempt cap before escalating |
| **UX Reviewer** | Observes every flow through Nielsen's 10 heuristics, calibrated to the actual target user persona |
| **Reporter** | Compiles a findings report and a graduation list at session end or on demand |
| **Orchestrator** | The human, in the loop at every judgment point |

## What's in this repo

- [`skill.md`](skill.md) — the `/test` Claude Code skill (the invocation prompt that runs the whole session)
- [`ARCHITECTURE.md`](ARCHITECTURE.md) — full design doc: the dual architecture, agent roles, session state schema, LangGraph upgrade path
- [`docs/orchestrator-protocol.md`](docs/orchestrator-protocol.md) — the human-in-the-loop rules and escalation patterns

## How to use it

### Prerequisites

- [Claude Code](https://claude.com/claude-code) CLI installed
- [Playwright MCP server](https://github.com/microsoft/playwright-mcp) configured in Claude Code
- A target web application accessible from your machine

### Setup

1. Drop `skill.md` into your Claude Code skills directory:
   ```
   ~/.claude/skills/test.md
   ```
   (or your project-specific skills location)

2. Create a context file for your platform under your vault or workspace:
   ```
   <your-vault>/<platform>/Context.md          # tech stack, credentials, hosting notes
   <your-vault>/<platform>/Test Plan v1.md     # what's in scope, expected behaviours
   <your-vault>/<platform>/Requirements v1.md  # feature requirements (if any)
   <your-vault>/<platform>/Session State.md    # written by the skill, picks up where you left off
   ```

3. Adjust the routing in `skill.md` Step 1 to point at your context files.

### Run a session

In Claude Code:

```
/test <platform>
```

The skill walks through:
- Discovery (Scout → Analyst → Planner)
- Execution loop (Designer → Executor → Triage / UX Reviewer)
- Area checkpoints (human decides continue / adjust / stop)
- Reporter (findings + Mode 2 graduation list)

### Generate Mode 2 specs

After Discovery, take the graduation list to Playwright CLI agents:

```bash
npx playwright planner "test the <flow-name> flow"
npx playwright generator specs/<flow>.md
npx playwright test
```

Generated `.spec.ts` files live in version control. Re-run on every deploy. Zero tokens.

## Pause / resume

If you say "pause" or "stepping away" mid-session, the skill writes a full session state to `Session State.md` — current position, open questions, next case, unresolved escalations. To resume, run `/test <platform>` again. The skill reads the state file and picks up from where it stopped.

## What's deliberately not here

- **A specific platform's results** — examples and screenshots live in the testing sessions, not in this repo
- **A LangGraph implementation** — v1 is Claude Code + Playwright MCP. The architecture is LangGraph-compatible; v2 will lift the agent boundaries and state schema directly. See `ARCHITECTURE.md` for the upgrade path
- **A CI/CD integration agent** — Mode 3 is documented in the architecture, not yet built. Triggered by PRs, generates specs from diffs, runs in CI. Needs GitHub Actions webhooks + git diff agent

## Background

This framework grew out of two earlier experiments:
- AI-assisted GxP validation testing in regulated environments
- AI-assisted Playwright automation for a SaaS application

Both worked, but both had the same shape: one AI alongside one tester. The next step was distributing the work — and keeping the human at the judgment points.

## License

MIT — see [LICENSE](LICENSE).
