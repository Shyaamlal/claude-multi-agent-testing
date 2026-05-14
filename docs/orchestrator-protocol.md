# Orchestrator Protocol

The framework only works if the human stays in the loop at the right points. Too involved and the agents are wasted; too detached and the findings drift. This doc spells out where the orchestrator must show up — and where they must stay out.

## What the Orchestrator does

| Decision | Who | Why |
|---|---|---|
| Severity (High / Medium / Low) on a UX finding | Human | Severity is a judgment call about real users, not a classification task |
| Whether an ambiguous failure is a real bug | Human | When Triage gives up after 2 attempts, the call is yours |
| What test area to do next after a checkpoint | Human | Continue, adjust scope, or stop — the agent doesn't know your priorities |
| Whether a new finding changes the queue | Orchestrator (you) prompting the Planner agent | The Planner re-plans, but you decide whether to ask it to |
| When to escalate mid-run to a developer | Human | "Critical bug, pause to file?" is your call — the agent only flags |

## What the Orchestrator does NOT do

| Action | Who | Why |
|---|---|---|
| Execute test cases | Test Executor | The Executor runs through Playwright MCP; you watch and decide on observations |
| Classify failures | Triage Agent | Triage has 2 attempts and a fixed taxonomy. Don't override unless ambiguous |
| Design test steps | Test Designer | Steps come from the live UI, derived 2-3 cases ahead. You shouldn't be writing them |
| Compile the final report | Reporter | Reporter is a spawned agent at session end. You review the output |

## Escalation patterns

Every agent that escalates does it the same way: **one specific question with context**.

Examples (good):
- "A2-05 Deactivate Client — the Off button triggers a native browser confirm dialog. Accepting works but I can't auto-snapshot the modal. Continue with the action, or stop here?"
- "OBS-07 — Client UI has no deactivate-user button. Likely Super-Admin-only by design. Flag in report, or worth testing as a bug?"

Examples (bad — don't accept these):
- "I noticed something weird. Want to see?" (vague — make the agent name what)
- "Here are 17 observations, what do you think?" (dump — make the agent pick the one that needs your input)

If an agent dumps instead of asking, push it back: "Give me the one focused question."

## Pause / resume cadence

You don't have to finish a session in one sitting. Say "pause" or "stepping away" and the framework writes a `Session State.md` with:
- Current area + cases completed/remaining
- Findings count and the list of open ones
- Queue of the next 3 cases to run
- Any unresolved escalations

To resume: `/test <platform>` reads the state file and picks up. The agents re-orient from the file, not from conversation memory.

## When to bail

If after 3+ escalations on the same finding the answer still isn't clear, stop running test cases and call a triage discussion separately. The framework is for execution; deep ambiguities need conversation, not test runs.

## Pattern: orchestrator as accountability layer

The point of keeping the human in the loop isn't to slow things down — it's to keep the framework honest. Specialist agents are very good inside their bounded task and bad outside it. They don't know:
- Which finding really matters to the business
- What level of risk is acceptable for the current release
- Whether a particular UX issue affects your actual users vs. a hypothetical one
- When scope should expand or contract based on what just got found

You do. So you stay in the loop at exactly those points and nowhere else.
