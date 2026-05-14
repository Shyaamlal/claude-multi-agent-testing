# Multi-Agent Testing Architecture

**Version:** 2.0
**Applies to:** Any web application

---

## Core Principle

Testing has two distinct modes. Mixing them is the most common mistake in test automation.

**Mode 1 — Discovery** is for understanding. New features, first passes, UX review, ambiguous behaviour. Human-in-the-loop, adaptive, judgment-driven. Output: findings and a list of flows stable enough to automate.

**Mode 2 — Regression** is for verifying known behaviour. Flows that passed cleanly in Discovery, automated as `.spec.ts` files via Playwright CLI agents. Runs from VS Code or terminal with zero tokens and no Claude session needed.

A flow graduates from Mode 1 to Mode 2 when:
- It passed cleanly in at least one Discovery session
- Expected behaviour is clearly defined
- UI elements are stable enough to trust selectors

The Reporter flags graduation candidates at the end of every Discovery session.

---

## The Dual Architecture

```
DISCOVERY (Mode 1)                    REGRESSION (Mode 2)
─────────────────────────────         ──────────────────────────
Tool: Playwright MCP                  Tool: Playwright CLI Agents
Session: Claude Code                  Session: Terminal / VS Code / CI
Tokens: Yes                           Tokens: Zero (after generation)
Human: In the loop                    Human: Reviews results
Output: Findings + graduation list    Output: Pass/fail on known behaviour
When: New features, exploration       When: After every deploy
```

When Mode 2 finds a regression → brings it back to Mode 1 for investigation.

---

## The Eight Agents (Mode 1 — Discovery)

### Orchestrator
Coordinates the session. Runs in the main Claude Code session throughout.

- Maintains `Session State.md`
- Assigns intents to specialist agents
- Receives findings and decides: auto-classify or escalate
- Runs end-of-area checkpoints with the human
- Adapts execution order based on findings
- Calls Reporter on demand or at session end

Does not: execute tests, judge UX severity, or classify failures independently.

---

### Architecture Scout
Runs once at session start. Reads the **live application**, not documentation.

- Navigates as each role — maps accessible pages, available actions, nav structure
- Identifies seams: role boundaries, data handoffs, state transitions
- Flags discrepancies between what docs describe and what the app shows
- Output: living system map that all other agents reference

This is the layer Playwright's own agents skip — they assume you already know what to test. The Scout answers that before any test intent is written.

---

### Requirements Analyst
Reads what exists — test plan, requirements docs, session state — then asks targeted questions about what's unclear or missing.

Conversational facilitator, not a document parser. Its job is arriving at shared understanding of what PASS looks like before execution begins.

- Input: existing documentation + Architecture Scout output
- Process: identifies gaps and ambiguities, asks specific questions to the human
- Output: agreed acceptance criteria for each area in scope

---

### Test Planner
Decides execution order. Stays active throughout — not just at session start.

- Input: Requirements Analyst output + current test plan state
- Output: prioritised test queue, updated after each area checkpoint
- Priority logic: fix verification first → untested critical flows → end-to-end chain → regression spot-check
- After each area checkpoint: replans based on findings

---

### Test Designer
Turns intent into executable steps. Works 2-3 cases ahead — never the full area upfront. This is what keeps the session adaptive rather than waterfall.

- Input: intent statement from Test Planner (e.g. "verify the Add User flow works end-to-end")
- Process: explores the relevant section of the live app, derives steps from what actually exists — not from memory
- Output: specific steps with URL, credentials, form values, expected result
- Verifies elements exist in the live app while designing (same pattern as Playwright's Planner)

Intent-driven, not script-driven. When the UI changes, the intent stays valid and steps are re-derived. Scripts break silently.

---

### Test Executor
Drives Playwright MCP. One test case at a time. Observes and reports — does not interpret.

- Input: test steps from Test Designer
- Output: what actually happened — pass/fail, unexpected behaviour, raw observations
- Uses accessibility tree locators (`getByRole`, `getByLabel`, `getByText`) — not CSS selectors
- Stops and reports to Orchestrator if something unexpected happens mid-step
- Black-box mode: browser only, no source code access. What you see is what a real user sees.

---

### Triage Agent
Activated on every failure or anomaly. Classifies before anything reaches the human.

Four outcomes:
- **Real bug** — system behaviour is clearly wrong
- **By design** — matches known platform constraints
- **Automation artifact** — behaviour that only occurs due to programmatic interaction, not reproducible by a real user. Verify manually before logging.
- **Ambiguous** — cannot classify confidently → escalate to human with one focused question

Guardrail: 2 attempts to reproduce or diagnose. If still unclear after 2 attempts, classify as ambiguous and escalate. Never loop indefinitely — same pattern as Playwright's Healer.

---

### UX Reviewer
Observes every flow the Test Executor runs. Evaluates through Nielsen's 10 Usability Heuristics.

- Findings logged separately from functional bugs
- Severity judgment (High/Medium/Low) escalated to human — UX severity is a judgment call
- Persona: use the actual target user for the application under test, not a generic QA lens

Functional bugs go to the developer queue. UX findings are a conversation with the product owner.

---

### Reporter
Compiles findings on demand or at session end.

Two outputs:
1. **Findings report** — functional bugs + UX findings in structured format
2. **Graduation list** — flows that passed cleanly and are candidates for Mode 2 spec file generation

The graduation list is what connects Mode 1 to Mode 2. Without it, Discovery sessions produce findings but never build toward automation.

---

## Execution Flow

```
SESSION START
│
├── Architecture Scout → system map of live app
├── Requirements Analyst → targeted questions → agreed acceptance criteria
├── Test Planner → priority queue
│
├── AREA LOOP
│   ├── Test Designer → 2-3 intent-driven test cases (live element verification)
│   │
│   ├── TEST CASE LOOP
│   │   ├── Test Executor → runs case (headed, black-box), raw observations
│   │   ├── Triage Agent → classify (2 attempts max)
│   │   │     ├── Confident → auto-log, next case
│   │   │     └── Ambiguous → one focused question to human
│   │   ├── UX Reviewer → logs UX observations from same flow
│   │   │     └── Severity judgment → escalate to human
│   │   └── Orchestrator → did findings change what we test next?
│   │         ├── Yes → Test Designer adjusts next batch
│   │         └── No → continue queue
│   │
│   └── AREA CHECKPOINT
│         "Area X done: N PASS, N FAIL, N NEW. Continue / adjust / stop?"
│
├── End-to-end chain (when component areas verified)
│
└── Reporter → Findings report + Graduation list for Mode 2
```

---

## Human-in-the-Loop Protocol

| Level | Trigger | What's needed |
|-------|---------|---------------|
| **Auto** | Confident Triage classification | Nothing — logged and continue |
| **Focused question** | Ambiguous failure · UX severity · unexpected new behaviour | One answer, one question |
| **Area checkpoint** | End of each test area | Continue / adjust scope / stop |

Rule: every escalation is a single specific question with context. No observation dumps.

---

## Session State Schema

Written to `Session State.md` after every test case.

```
## Session — [DATE]

### Current position
- Area: [name]
- Cases completed / remaining: [n / n]
- Last: [ID] — [PASS/FAIL/NEW]

### Findings
- Functional bugs: [n]
- UX findings: [n]
- Fix-verified: [n]
- Graduation candidates: [list]

### Queue (next 3)
- [ID] — [intent]

### Open questions
- [any unresolved escalations]
```

---

## Mode 2 — Regression Suite Generation

After a Discovery session, take the graduation list to Playwright CLI agents:

```bash
npx playwright planner "test the [flow name] flow"
npx playwright generator specs/[flow].md
```

Generated `.spec.ts` files live in version control. Run anytime:

```bash
npx playwright test
```

No Claude session. No tokens. Immediate pass/fail signal on known behaviour.

When a spec fails after a deploy → bring it back to Mode 1 to investigate.

---

## Mode 3 — CI/CD Integrated Testing (Future)

A third mode beyond Discovery and Regression. Not yet built — documented so the architecture is complete.

**Trigger:** A pull request is raised in the codebase.

**Flow:**
1. Testing agent analyses the PR diff — what changed, what it touches, what could break
2. Posts testing feedback on the PR before approval
3. On PR approval — test scripts generated for the changed flows
4. Scripts pushed to CI/CD, executed automatically
5. Results posted back to the PR

**Gaps today:** GitHub Actions webhooks, agent that reads git diff, CI/CD pipeline.

**Relationship to Modes 1 and 2:**
Mode 1 discovers → Mode 2 automates proven flows → Mode 3 triggers Mode 2 automatically on every code change.

LangGraph is the right infrastructure here — triggered, stateless, CI/CD-compatible. Build after Mode 2 is running cleanly.

---

## What Playwright Agents Cover (Don't Rebuild)

| Playwright Agent | What it does | Our equivalent |
|---|---|---|
| Planner | Intent → Markdown test plan | Test Designer pattern |
| Generator | Markdown → `.spec.ts` files | Mode 2 generation |
| Healer | Failing test → fix or skip | Triage Agent pattern |

Use Playwright CLI agents directly for Mode 2 generation. The Planner/Generator/Healer are battle-tested — no need to replicate the logic.

The value layer of this framework is everything they don't cover: Architecture Scout, Requirements Analyst, Test Planner, UX Reviewer, Orchestrator, and the dual-mode framework itself.

---

## LangGraph — Upgrade Path (v2)

When this framework becomes a reusable product:
- Each agent becomes a LangGraph node with defined state in/out
- `interrupt()` replaces the manual escalation protocol
- Checkpointing replaces `Session State.md`
- Parallel branches run UX Reviewer and Test Executor simultaneously

v1 is designed LangGraph-compatible. Agent boundaries and state schema lift directly.
