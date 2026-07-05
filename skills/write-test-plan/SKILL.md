---
name: write-test-plan
description: Use when user provides stories or feature requirements and needs a test plan written, before writing implementation plans
---

## When to Use

- User provides stories, feature requirements, or a brief description of what to build
- After `story-analysis`, BEFORE `writing-plans`
- The output test plan becomes the acceptance criteria for implementation

## Input

Stories can come in any form:
- A CSV file with short story descriptions
- A list of requirements in text
- A feature brief or spec document

Read and understand all stories before writing test cases.

**Input validation**: If the input is a story-analysis document, check that it includes a **Behavioral Dimensions** section. If missing, flag to the user that dimensions were not analyzed and ask whether to proceed or revisit story-analysis first.

## Output Format

Write the test plan as **self-contained HTML with a left-side sticky TOC** — copy the structure from the shared skeleton at `../feature-workflow/references/plan-doc-skeleton.html` (in the `fatsecret-workflow` plugin). Render the cases as one or more `<table>`s grouped by feature area (see Grouping below); use the `--pass-*` / `--warn-*` / `--gate-*` CSS vars from the skeleton for status cells if you include a result column.

Use `example-test-plan.csv` (in this skill's directory) as the reference for the **column structure and the multi-row step pattern** — map those columns to table headers and the step rows to table rows.

**CSV alternative:** If the project's CLAUDE.md asks for CSV, or QA needs to import the plan into a spreadsheet, emit `test-plan.csv` matching `example-test-plan.csv` instead. HTML is the default; CSV is the explicit-request fallback.

**Unit Test Coverage** is rendered as its own `<table>`, kept separate from the manual QA cases — they are different artifacts (see "Unit Test Coverage" below).

## Writing Guidelines

### Test Case Coverage

**Coverage is driven by the story-analysis Behavioral Dimensions**, not by a fixed checklist. For each story/feature, generate cases for every dimension the feature actually passes through or is gated by. The recurring dimensions in this app — include each ONLY when the feature is reachable through it or behaves differently under it:
- **Happy path**: Normal user flow works correctly (always)
- **UI verification**: Visual elements are present and correct (always for UI work)
- **Entry points**: The ways this feature is actually reachable (tabs, buttons, notifications, widgets, deep links, 3D Touch) — enumerate the real ones, don't template all of them
- **Account types**: Member vs Premium Member — only if behavior or availability differs
- **Market restrictions**: only if the feature is market-gated
- **Edge cases**: Empty states, no results, first-time users, returning users
- **Navigation**: Back button, close button, dismissing flows
- **Persistence**: State preserved after navigation, reinstall, app update
- **Data integrity**: Items saved/logged correctly

Irrelevant-dimension cases dilute the plan and waste QA time — coverage breadth comes from the feature's real dimensions, not from checklist completeness.

### Actionability — every case must be executable as written

Each manual case states:
1. **Preconditions** — account type, app state, data setup needed before starting
2. **Steps** — concrete numbered actions ("tap Add Food on the Diary Breakfast row"), not intentions ("verify logging works")
3. **Expected result** — the observable outcome (what appears / changes / persists, exact text where it matters)

A QA engineer must be able to execute the case without asking what it means or how to reach the screen. "Verify X works correctly" without steps and an observable result is not a test case.

### Naming Convention

Titles should follow: `[N] Verify [specific observable outcome] when [trigger/condition]`

Examples:
- `[1] Verify Food logging bottom sheet is opening when Add Food is tapped from Diary meal type`
- `[9] Verify UI of the Launcher bottom sheet`
- `[39] Verify Food is recorded correctly in Diary`

### Grouping

Order test cases logically by feature area:
1. Entry points and navigation
2. UI verification
3. Core functionality
4. Edge cases and error states
5. Account/market-specific behavior
6. Persistence and data integrity

### Unit Test Coverage

Beyond the manual QA cases above, the test plan defines which logic gets **automated unit tests**. Keep this in a **separate section/table** from the manual cases — the two are different artifacts: manual cases are executed by a human against the running app; unit tests are code written alongside the implementation (TDD) and run by `xcodebuildmcp test`.

Identify the **deterministic, non-UI units** worth covering:
- Pure functions and computed transforms
- View models / presenters (the logic, not the SwiftUI view)
- Parsers, serializers, mappers
- Business rules and validation
- State-transition logic (flag/counter changes, upgrade-path logic)

For each, record: the unit under test → the key behaviors / boundaries to assert → which Item (from story-analysis) it backs.

**Out of scope:** UI tests and snapshot tests. The unit test target is **`Calorie Counter Tests`** (already present in the project).

This section is the source `writing-plans` reads to decide which task gets which unit tests (TDD, per task).

## Coverage Challenge (Codex devil's advocate) — mandatory

**Type: Rigid gate. Do not skip, do not rationalize past it.** Before the test plan is finalized and written out, the coverage MUST survive an adversarial review by Codex. Run it on the **draft coverage** — both the manual QA cases AND the Unit Test Coverage section — before saving `test-plan.html`:

1. Open a Codex session with `mcp__codex__codex`, configured `cwd` = project root, `sandbox: read-only`, `approval-policy: never`.
2. Frame Codex as a **picky devil's advocate** whose only job is to attack the coverage. Send it the draft cases + unit-test scope plus the story-analysis / design inputs, and ask it to find:
   - **Manual QA gaps**: missing entry points, account/market dimensions, edge/empty states, upgrade-path scenarios, navigation/persistence holes
   - **Unit-test gaps**: logic branches with no assertion, untested boundary values, error paths, false confidence (a "covered" unit whose real risk lives elsewhere)
   - **Wrong/redundant cases**: cases that are duplicated, assert the wrong thing, or cover a dimension the feature doesn't actually pass through
   - **Non-actionable cases**: cases a QA engineer could not execute as written (missing preconditions, vague steps, unobservable expected results)
3. Evaluate each challenge with project context — accept real gaps, push back (with reasoning) on speculative ones. Iterate via `mcp__codex__codex-reply`; keep the thread.
4. Fold the agreed gaps into the coverage. Only then proceed to write `test-plan.html`.

If `mcp__codex__codex` is unavailable, stop and ask the user to enable it — do NOT finalize the test plan by skipping this gate.

| Thought | Reality |
|---|---|
| "Coverage already looks complete" | Then Codex finds nothing and the gate costs one round. Run it. |
| "Auto mode means I can skip this" | Auto mode does not disable a Rigid gate. |
| "This feature is small" | Small features have the cheapest, fastest challenge. Run it anyway. |

## External Authoritative QA Plan

If the QA team maintains its own test plan for this feature (a spreadsheet, an artifact link, a case-management tool), **theirs is authoritative**. Ask the user whether one exists before finalizing. When it does: map this plan's cases to theirs, reconcile gaps in both directions (cases they have that this plan missed become additions here; cases here that they lack get flagged to the user for hand-off), and record the mapping — do not maintain two divergent plans in parallel. Step 7a verification then runs against the authoritative plan.

## Finalize & Hand Off

Reach this point only after the mandatory **Coverage Challenge** above has resolved against the draft coverage.

1. Save the test plan (with the Unit Test Coverage section) to the feature's output directory (see feature-workflow Output Directory rule) as `test-plan.html` (or `test-plan.csv` per the fallback above)
2. Present a summary to the user: total manual cases, unit-test units covered, coverage areas, and the gaps the Codex challenge surfaced + how they were resolved
3. Ask user to review and approve before proceeding to `writing-plans`
4. The approved test plan becomes the acceptance criteria for implementation — its Unit Test Coverage section drives the per-task TDD steps in `writing-plans`
