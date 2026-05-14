# /test — Multi-Agent Testing Skill

Run a multi-agent testing session for the specified platform.

**Usage:** `/test <platform>` — e.g. `/test acme-app`

If no platform is given, ask: "Which platform are we testing today?"

---

## ARCHITECTURE REFERENCE

Before doing anything, read: `ARCHITECTURE.md` (this repo's design doc).
This is the full framework. The steps below implement it.

---

## STEP 1 — LOAD CONTEXT (silent, no output yet)

Based on the platform argument, load the following from the user's vault / workspace:

- Platform context: `<workspace>/<platform>/Context.md` (tech stack, credentials, hosting notes)
- Test plan: `<workspace>/<platform>/Test Plan v1.md` (what's in scope, expected behaviours)
- Requirements: `<workspace>/<platform>/Requirements v1.md` (feature requirements, if any)
- Session state: `<workspace>/<platform>/Session State.md` (picks up where you left off)
- Test environment: the platform's live URL

Also read today's daily note (if the user uses one) for any session handoff from a previous run.

**Customise the routing logic for each platform** — every project has its own context file, test plan, requirements, and session state. The framework logic doesn't change.

---

## STEP 2 — SESSION MODE DECLARATION

State explicitly before proceeding:

- **Session type:** Verify (re-testing known areas after fixes), Explore (new/untested areas), or Both
- **Mode:** Discovery (Mode 1 — this skill)
- **Browser:** Headed (visible) — black-box testing. Browser tools only. Do not read source code.

Ask the user to confirm or adjust the session type before continuing.

---

## STEP 3 — ARCHITECTURE SCOUT

Spawn as a focused agent (Sonnet 4.6):

> "You are the Architecture Scout for a multi-agent testing session. Navigate the live application at [URL]. Log in as each available role. For each role, map: what pages are accessible, what actions are available, what the navigation structure looks like. Identify the seams: role boundaries (can Role A access Role B's pages?), data handoffs (where does one role's output become another role's input?), and state transitions (zero states, deactivation, edge conditions). Note any discrepancies between what the documentation describes and what the live app shows. Do not read source code. Output: a structured system map."

Present the system map to the user. Ask: "Does this match your understanding, or have you spotted anything unexpected?"

---

## STEP 4 — REQUIREMENTS ANALYST

Read the test plan, requirements doc, and session state. Then ask targeted questions about what is unclear or missing — specifically:

- For untested cases: what does PASS actually look like?
- For previously BLOCKED cases now unblocked: what changed, and what should we expect?
- For ambiguous expected results: what is the business intent?

This is a conversation, not a document summary. Ask one focused question at a time. Stop when acceptance criteria are clear for the session's scope.

Write agreed acceptance criteria to `Session State.md` before proceeding.

---

## STEP 5 — TEST PLANNER

Based on requirements analysis and current test plan state, set the execution queue.

**Default priority order:**
1. Verify fixes — re-run previously BLOCKED or FAILED cases first
2. Untested cases in critical areas
3. End-to-end chain (when component areas verified)
4. Regression spot-check on previously PASSED cases

Present the queue to the user: "Here's the execution order for today. Anything to adjust?"

Update `Session State.md` with the confirmed queue.

---

## STEP 6 — EXECUTION LOOP

Repeat for each test area:

### Test Designer (intent-driven, 2-3 cases ahead)
Design the next 2-3 test cases from intent — not from memory. For each:
- Navigate to the relevant section of the live app using Playwright MCP
- Verify elements exist before writing the steps
- Output: specific steps with URL, credentials, form values, expected result

Intent over script. If the UI has changed, re-derive steps from what's actually there.

### Test Executor (Playwright MCP, headed)
Run one test case at a time. Report raw observations — what actually happened. Do not interpret or classify. If something unexpected occurs mid-step, stop and report to the Orchestrator before continuing.

**Playwright MCP tools to use:**
- `browser_navigate` — navigate to URLs
- `browser_snapshot` — read accessibility tree (preferred over screenshots for element location)
- `browser_click`, `browser_fill_form`, `browser_type` — interact with elements
- `browser_take_screenshot` — capture visual state at every meaningful step: before form submission, after result, on any anomaly
- Use `getByRole`, `getByLabel`, `getByText` locator patterns — not CSS selectors

**Write to the Test Execution Log after every test case:**
`[DATE TIME] | [ID] | [Area] | [PASS/FAIL/ANOMALY] | [one-line description] | [screenshot filename if taken]`

This is the timestamped audit trail. Every case gets an entry — pass or fail.

### Triage Agent (inline reasoning)
For every failure or anomaly:
1. Attempt to reproduce (max 2 attempts)
2. Classify:
   - **Real bug** — system behaviour is clearly wrong
   - **By design** — matches known constraints
   - **Automation artifact** — only occurs via programmatic interaction; verify manually before logging
   - **Ambiguous** — cannot classify after 2 attempts → escalate
3. **Log the reasoning** — not just the classification. For every finding, write: what was observed, what was expected, why this classification was chosen. This is the audit trail that makes findings defensible if disputed later.
4. **Take a screenshot for every finding** — `browser_take_screenshot` required for every bug and UX finding, not just report sections. No screenshot = no logged finding.

If ambiguous: bring ONE focused question to the user. Never dump raw observations.

### UX Reviewer (inline, every flow)
While executing each flow, observe through Nielsen's 10 Usability Heuristics:
- Error messages — informative or generic?
- Form validation — immediate and clear?
- Navigation — can the user get lost?
- Consistency — do similar actions behave similarly across roles?
- Recovery — when something fails, can the user recover?

Use the actual target user persona for the platform — not a generic QA lens.

Log UX findings separately from functional bugs. Escalate severity judgments (High/Medium/Low) to the user.

### Update Session State after every test case:
```
## Session — [DATE] [TIME]
### Current position
- Area: [name] | Cases completed/remaining: [n/n] | Last: [ID] — [result]
### Findings so far
- Functional bugs: [n] | UX findings: [n] | Fix-verified: [n]
### Queue (next 3)
- [ID] — [intent]
### Open questions
- [any unresolved escalations]
```

### Area Checkpoint (end of each area)
Pause and present:
> "Area [X] complete: [N] PASS, [N] FAIL, [N] NEW findings, [N] UX observations.
> [Brief summary of what we found.]
> Continue to [Area Y], adjust scope, or stop?"

**Log the human decision** with a timestamp. This is the audit record of judgment calls.

Wait for response before continuing.

---

## HUMAN-IN-THE-LOOP PROTOCOL

| Trigger | Action |
|---------|--------|
| Confident Triage classification | Auto-log, continue — no escalation |
| Ambiguous failure | One focused question with context |
| UX severity judgment | Ask: High / Medium / Low? |
| Unexpected new behaviour (not in test plan) | "This wasn't in scope — is it worth logging?" |
| Critical bug mid-run | "This is high severity. Continue or pause to report to [developer]?" |
| End of area | Area checkpoint (see above) |

Rule: every escalation is one specific question. Never narrate. Never dump.

**The human operator is the Orchestrator — not an approver.**
Before the Executor runs the next test case, confirm you understand what the last agent found and why the next step follows from it. If you can't explain it, ask before proceeding. Control of what's happening across the board stays with you — not with the agents.

---

## STEP 7 — END-TO-END CHAIN

When all component areas are verified, run the end-to-end scenario across all user roles. This is the prize test. Only run when the individual pieces are confirmed working.

---

## STEP 8 — REPORTER

Call at end of session or on demand ("generate report").

Spawn as a focused agent (Sonnet 4.6):

> "You are the Reporter for a testing session. Based on the session state and findings logged during this run, produce two outputs:
> 1. Findings Report — functional bugs (ID, area, description, severity, status: new/regression/fix-verified) and UX findings (ID, heuristic violated, description, severity, recommendation) in a structured format matching prior reports.
> 2. Graduation List — test cases that passed cleanly in this session and are candidates for Mode 2 Playwright CLI spec file generation. A case qualifies when: it passed cleanly, expected behaviour is clearly defined, UI elements are stable."

Write the report to `<workspace>/<platform>/Test Report [DATE].md`.
Update the test plan status table with results from this session.
Log the session in today's daily note (if used) under `## Claude's Log`.

---

## PAUSE PROTOCOL

If the user says "pause" or "stepping away":
1. Write full session state to `Session State.md` — current position, open questions, next case, any unresolved escalations
2. Note in today's daily note (if used): "Testing session paused — see Session State.md to resume"

To resume: `/test <platform>` — the skill reads `Session State.md` and picks up from where it left off.

---

## MODEL SELECTION

- **Orchestrator** (this session): current model
- **Spawned agents** (Architecture Scout, Reporter): Sonnet 4.6 — fast, bounded tasks
- **Playwright MCP**: always uses the current session's model

---

## ADDING A NEW PLATFORM

This skill is designed to work with any web platform. To add a new platform:
1. Create a context file at `<workspace>/<platform>/Context.md`
2. Create a test plan at `<workspace>/<platform>/Test Plan v1.md`
3. Add platform routing to Step 1 of this skill

The framework logic does not change. Only the context changes.
