---
name: write-test-plan
description: Use when user provides stories or feature requirements and needs a test plan written, before writing the execution plan
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

**Language: English.** The plan is executed by QA and uploaded to TestSecret as shared cases — write it in English even when the working conversation is in another language. This covers case titles, preconditions, steps, and expected results.

Write the test plan as **self-contained HTML with a left-side sticky TOC** — copy the structure from the shared skeleton at `../feature-workflow/references/plan-doc-skeleton.html` (in the `fatsecret-workflow` plugin). Render the cases as one or more `<table>`s grouped by feature area (see Grouping below); use the `--pass-*` / `--warn-*` / `--gate-*` CSS vars from the skeleton for status cells if you include a result column.

Use `example-test-plan.csv` (in this skill's directory) as the reference for the **column structure and the multi-row step pattern** — map those columns to table headers and the step rows to table rows.

**CSV alternative:** If the project's CLAUDE.md asks for CSV, or QA needs to import the plan into a spreadsheet, emit `test-plan.csv` matching `example-test-plan.csv` instead. HTML is the default; CSV is the explicit-request fallback.

**Verification Plan** is rendered as its own `<table>`, kept separate from the manual QA cases — they are different artifacts (see "Verification Plan" below).

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

### Verification Plan

Beyond the manual QA cases above, the test plan defines **how every deliverable is verified**: one table, one row per deliverable — deliverable → **criteria** (the observable result that proves it works) → **method**. This table is confirmed with the human before the plan is approved, is what Step 6 tasks execute, and is what `review-task` re-runs.

**Verification is chosen by risk, not by structure — and e2e beats unit.** Unit tests cost build minutes and are written ONLY where the answer itself can silently be wrong. Everything else is verified at the boundary it ships at. Methods, in order of preference:

- **Script + fixture** — view models and anything with server interaction. A runnable script committed to `verification/<feature>/` in the repo drives the flow; with Proxyman available, assert on the captured request/response; without it, inspect the script's input and output directly. The script and its recorded fixture ARE the regression artifact — a throwaway script that isn't committed counts as unverified.
- **Review-only** — mapping / assembly / pass-through model code: a Codex (or advisor) review is the verification; the recorded review verdict is the evidence. No test written.
- **Unit test** (`Calorie Counter Tests`) — ONLY for wrong-answer-is-a-bug logic: calculations, unit/portion conversion, percentage and date arithmetic, state transitions (flag/counter changes, upgrade paths). A mapper or view-model wiring does NOT qualify. Asserted on observable results from an independent source of truth, written failing-first.
- **Integration test** (real store, existing pattern `FoodJournalSaveContractTests`) — only when the deliverable itself is a persistence contract (save → release → reload → intact; cancel leaves stored state unchanged).
- **Manual QA** — visual/UX behavior; it lives in the manual cases table above, not here.

One method per row (the cheapest that can observe the failure). The goal is that every behavior which, if wrong, would silently give the user bad data or a broken flow has exactly one verification that catches it.

For each row, record: the behavior → the assertion (expected value from an independent source: the design's input → output example, the API/data contract, a known-good literal — never recomputed the way the code will compute it) → **Prevents**: the concrete real error this test would catch if it did not exist → which Item (from story-analysis) it backs. **A row whose Prevents cannot be filled with a specific error is dropped**, not kept "for coverage".

**Not in this section:** getter/setter or pass-through checks, tests that only assert a method was called, behavior the framework or language already guarantees, UI tests and snapshot tests. **Presentation math is not a business rule, even once it has been lifted into a pure struct:** layout geometry (origins, widths, text scale, collision/overflow), colour interpolation, corner roles, and source-grepping purity guards stay out of the Unit layer — the Figma comparison in `review-task` and manual QA own them. The line is: a wrong answer here gives the user bad *data* or a broken *flow* → Unit; a wrong answer here gives the user a misplaced or miscoloured *pixel* → not here.

This section is the source `writing-plans` reads to assign each task its Verification rows.

## Coverage Challenge (devil's advocate) — mandatory

**Type: Rigid gate. Do not skip, do not rationalize past it.** Before the test plan is finalized and written out, the coverage MUST survive an adversarial review — by Codex when available, by a fresh-context subagent otherwise (see fallback below). Run it on the **draft coverage** — both the manual QA cases AND the Verification Plan — before saving `test-plan.html`:

1. Open a Codex session with `mcp__codex__codex`, configured `cwd` = project root, `sandbox: read-only`, `approval-policy: never`.
2. Frame Codex as a **picky devil's advocate** whose only job is to attack the coverage. Send it the draft cases + verification plan plus the story-analysis / design inputs, and ask it to find:
   - **Manual QA gaps**: missing entry points, account/market dimensions, edge/empty states, upgrade-path scenarios, navigation/persistence holes
   - **Verification gaps**: a deliverable with no row, criteria that nothing can observe, a method too weak for its risk (review-only on calculation logic, a script that checks the happy path but not the failure the criteria name), a persistence flow with no integration row, a committed-fixture claim with no fixture
   - **Cost-only rows**: a Prevents that names no concrete error, a rule asserted at both layers, an expected value derived the way the implementation computes it, a row that would break on a harmless refactor, a presentation-math row (geometry, colour, corner role, purity guard) that the "Not in this section" rule already excludes
   - **Wrong/redundant cases**: cases that are duplicated, assert the wrong thing, or cover a dimension the feature doesn't actually pass through
   - **Non-actionable cases**: cases a QA engineer could not execute as written (missing preconditions, vague steps, unobservable expected results)
3. Evaluate each challenge with project context — accept real gaps, push back (with reasoning) on speculative ones. Iterate via `mcp__codex__codex-reply`; keep the thread.
4. Fold the agreed gaps into the coverage. Only then proceed to write `test-plan.html`.

**Codex unavailable → internal fallback, not a stop**: hand the same devil's-advocate brief (step 2's attack list, the draft coverage, the story-analysis / design inputs) to a **fresh-context subagent** (`general-purpose`, read-only) that has not seen how the draft was constructed. Evaluate and fold its challenges exactly as in steps 3–4, and record in the plan summary that the challenge ran on the internal fallback. The gate is on the adversarial pass, not on Codex specifically — it is never skipped.

| Thought | Reality |
|---|---|
| "Coverage already looks complete" | Then the challenger finds nothing and the gate costs one round. Run it. |
| "Auto mode means I can skip this" | Auto mode does not disable a Rigid gate. |
| "This feature is small" | Small features have the cheapest, fastest challenge. Run it anyway. |

## TestSecret — the authoritative case library

The team's manual QA cases live in **TestSecret** (the `testsecret` MCP server). Once this plan's cases are uploaded (see Finalize & Hand Off), the TestSecret copies are authoritative: Step 7a validates against them and records results there. If QA also maintains a separate artifact for this feature (a spreadsheet, a doc), reconcile it into this plan **before** upload — cases theirs has that this plan missed become additions here; cases here that theirs lacks get flagged to the user — so a single source of truth goes up, never two divergent plans.

## Finalize & Hand Off

Reach this point only after the mandatory **Coverage Challenge** above has resolved against the draft coverage.

1. Save the test plan (with the Verification Plan section) to the feature's output directory (see feature-workflow Output Directory rule) as `test-plan.html` (or `test-plan.csv` per the fallback above)
2. Present a summary to the user: total manual cases, verification rows per method (script / review-only / unit / integration), coverage areas, and the gaps the Codex challenge surfaced + how they were resolved
3. Ask user to review and approve — **present the Verification Plan as ONE table in ONE batch confirmation** (the criteria and method per deliverable are the human's call; they correct individual rows in their reply — never ask about rows one at a time) — before proceeding to `writing-plans`
4. **Upload the approved cases to TestSecret** (see below)
5. The approved test plan becomes the acceptance criteria for implementation — its Verification Plan drives each task's verify step in `writing-plans`

### Upload to TestSecret (after approval)

Upload the **manual QA cases only** — the Verification Plan stays in the repo (it drives implementation verification, not manual QA).

1. **Preflight**: call `whoami` on the `testsecret` MCP. If the server is unreachable, tell the user and record the deferred upload in the test plan — never silently skip. If the MCP is not configured at all, offer to set it up with the guide at https://claude.ai/code/artifact/656220b8-5cc5-46de-8880-e330839cbb62.
2. **Section**: `list_sections`; reuse the section named after the feature, or `create_section` for it.
3. **Re-run guard**: `list_cases` filtered by the feature tag. If cases from a previous upload exist, patch the changed ones via `bulk_update_cases` (match on `ref_id`) instead of creating duplicates.
4. **Payload** per case: `title` (case title with the `[N]` prefix stripped), `ref_id` = `<feature-slug>-<N>`, `section_id`, `preconditions`, `steps` (one `{description, expected}` per step row), `tags` = `["<feature-slug>"]`, `refs` = story/Figma links when available, `status` = `Draft` — QA owns the Draft → Ready → Approved promotion inside TestSecret.
5. **Upload**: `bulk_create_cases` (≤200 per call, all-or-nothing; generate the required `idempotency_key` with `uuidgen`).
6. **Write the mapping back**: add each case's TestSecret id to its row in `test-plan.html` — Step 7a's result write-back needs it.
7. Report to the user: how many cases went up, into which section, under what tag.
