---
name: write-test-plan
description: Use after the design is confirmed to write the feature's two test artifacts — the developer's Verification Plan (one row per deliverable stating how it is proven done, appended to design.html) and a draft QA test plan (manual cases in test-plan.html, uploaded to TestSecret as Draft for QA to review, edit and own).
---

## When to Use

- After the feature-workflow Step 3 design gate, BEFORE `writing-plans`
- Standalone: any confirmed design whose deliverables need verification criteria and a draft set of QA cases

## Two outputs, two owners

- **Verification Plan** — dev's own test considerations: how dev proves each deliverable done before it reaches QA. Appended to `design.html`.
- **Draft QA test plan** — manual cases written from the confirmed design, saved as `test-plan.html` and uploaded to TestSecret as **Draft**. It is a head start for QA, not the final word: from upload on QA owns the cases — they review, edit, add, delete and promote them (Draft → Ready → Approved) with their own method. QA's shared understanding of the decisions comes from the summary posted to the Shortcut stories at the Step 3 gate; the draft cases turn that summary into executable steps.

## Input

- The confirmed `design.html`: D-decisions, acceptance points, input → output examples, non-goals
- The story-analysis output when it exists. **Input validation**: check that it includes a **Behavioral Dimensions** section — case coverage and the coverage challenge need it. If missing, flag to the user and ask whether to proceed or revisit story-analysis first.

## Output

**Language: English** — the cases go into TestSecret as shared records; case titles, preconditions, steps and expected results are all English, even when the working conversation is in another language.

- **Verification Plan**: a section appended to the feature's `design.html`, before the Execution Checklist, as one `<table>` using the shared skeleton's styling (`../feature-workflow/references/plan-doc-skeleton.html`).
- **Draft QA test plan**: `test-plan.html` in the feature's output directory — self-contained HTML with a left-side sticky TOC, copied from the same skeleton. Cases are `<table>`s grouped by feature area (see Grouping); each case has a title, preconditions, and numbered step rows of **action → expected result** — the same shape TestSecret stores (`preconditions`, `steps[{description, expected}]`).

## Verification Plan

One row per deliverable: deliverable → **criteria** (the observable result that proves it works) → **method** → **Prevents** → Item (from story-analysis). This table is confirmed with the human, is what Step 6 tasks execute, and is what `review-task` re-runs.

**Verification is chosen by risk, not by structure — and e2e beats unit.** Unit tests cost build minutes and are written ONLY where the answer itself can silently be wrong. Everything else is verified at the boundary it ships at. Methods, in order of preference:

- **Script + fixture** — view models and anything with server interaction. A runnable script committed to `verification/<feature>/` in the repo drives the flow; with Proxyman available, assert on the captured request/response; without it, inspect the script's input and output directly. That fallback only works when the script itself makes the call: when the criteria are about a request **the app itself sends**, only a capture can observe it — without Proxyman the row is reported UNVERIFIED and carried to Step 7a, never passed. The script and its recorded fixture ARE the regression artifact — a throwaway script that isn't committed counts as unverified.
- **Review-only** — mapping / assembly / pass-through model code: a Codex (or advisor) review is the verification; the recorded review verdict is the evidence. No test written.
- **Simulator check** — visual/UX deliverables: `review-task`'s UI verification against Figma is the check.
- **Unit test** (`Calorie Counter Tests`) — ONLY for wrong-answer-is-a-bug logic: calculations, unit/portion conversion, percentage and date arithmetic, state transitions (flag/counter changes, upgrade paths). A mapper or view-model wiring does NOT qualify. Asserted on observable results from an independent source of truth, written failing-first.
- **Integration test** (real store — follow the project's existing save-contract tests) — only when the deliverable itself is a persistence contract (save → release → reload → intact; cancel leaves stored state unchanged).

One method per row — the cheapest that can observe the failure. The goal is that every behavior which, if wrong, would silently give the user bad data or a broken flow has exactly one verification that catches it. **Criteria** take their expected values from an independent source — the design's input → output examples, the API/data contract, a known-good literal — never recomputed the way the code will compute them. **A row whose Prevents cannot be filled with a specific real error is dropped**, not kept "for coverage".

**Not in this section:** getter/setter or pass-through checks, tests that only assert a method was called, behavior the framework or language already guarantees, UI tests and snapshot tests. **Presentation math is not a business rule, even once it has been lifted into a pure struct:** layout geometry (origins, widths, text scale, collision/overflow), colour interpolation, corner roles, and source-grepping purity guards stay out of the Unit layer — the simulator check and QA own them. The line is: a wrong answer gives the user bad *data* or a broken *flow* → a verification row; a wrong answer gives the user a misplaced or miscoloured *pixel* → simulator check / QA.

This section is the source `writing-plans` reads to assign each task its Verification rows.

## Draft QA Test Plan

### Case coverage

**Coverage is driven by the story-analysis Behavioral Dimensions**, not by a fixed checklist. For each story/feature, write cases for every dimension the feature actually passes through or is gated by. The recurring dimensions in this app — include each ONLY when the feature is reachable through it or behaves differently under it:
- **Happy path**: normal user flow works (always)
- **UI verification**: visual elements present and correct (always for UI work)
- **Entry points**: the ways this feature is actually reachable (tabs, buttons, notifications, widgets, deep links) — enumerate the real ones, don't template all of them
- **Account types**: member vs premium, guest vs registered — only if behavior or availability differs
- **Market restrictions**: only if the feature is market-gated
- **Edge cases**: empty states, no results, first-time users, returning users
- **Navigation**: back, close, dismissing flows
- **Persistence**: state preserved after navigation, reinstall, app update
- **Data integrity**: items saved/logged correctly

Irrelevant-dimension cases dilute the plan and waste QA time — breadth comes from the feature's real dimensions, not from checklist completeness. The design's input → output examples and edge cases are the expected results; never invent behavior the design didn't decide.

### Actionability — every case executable as written

Each case states:
1. **Preconditions** — account type, app state, data setup needed before starting
2. **Steps** — concrete actions ("tap Add Food on the Diary Breakfast row"), not intentions ("verify logging works")
3. **Expected result** per step — the observable outcome (what appears / changes / persists, exact text where it matters)

A QA engineer must be able to run the case without asking what it means or how to reach the screen. "Verify X works correctly" without steps and an observable result is not a test case.

### Naming

Titles follow `[N] Verify [specific observable outcome] when [trigger/condition]` — e.g. `[1] Verify the food logging sheet opens when Add Food is tapped from a Diary meal row`.

### Grouping

Order cases by feature area: entry points and navigation → UI verification → core functionality → edge cases and error states → account/market-specific behavior → persistence and data integrity.

## Coverage Challenge (devil's advocate) — mandatory

**Type: Rigid gate. Do not skip, do not rationalize past it.** Before either artifact is finalized, both MUST survive an adversarial review — by Codex when available, by a fresh-context subagent otherwise (see fallback below):

1. Write both drafts first — the Verification Plan into `design.html`, the cases into `test-plan.html` — so the challenger can read them by path. Open a Codex session with the Codex CLI: `codex exec --sandbox read-only -C <project root> -o <reply file> "<brief>"`, continuing the same thread with `codex exec resume --last`.
2. Frame Codex as a **picky devil's advocate** whose only job is to attack the coverage. Point it at both drafts plus `design.html` and the story-analysis output, and ask it to find:
   - **Dimension gaps**: a Behavioral Dimension that changes behavior (account type, market, entry point, upgrade state, guest vs registered) with no row or case covering it
   - **Verification gaps**: a deliverable with no row, criteria that nothing can observe, a method too weak for its risk (review-only on calculation logic, a script that checks the happy path but not the failure the criteria name), a persistence flow with no integration row, a committed-fixture claim with no fixture
   - **Cost-only rows**: a Prevents that names no concrete error, a rule asserted twice, an expected value derived the way the implementation computes it, a row that would break on a harmless refactor, a presentation-math row the "Not in this section" rule already excludes
   - **Case gaps**: missing entry points, account/market dimensions, edge/empty states, upgrade-path scenarios, navigation/persistence holes
   - **Wrong or redundant cases**: duplicates, a case asserting something the design didn't decide, a dimension the feature doesn't pass through
   - **Non-actionable cases**: missing preconditions, vague steps, unobservable expected results
3. Evaluate each challenge with project context — accept real gaps, push back (with reasoning) on speculative ones. Iterate in the same thread (`codex exec resume --last`).
4. Fold the agreed gaps into both drafts.

**Codex CLI unavailable → internal fallback, not a stop**: hand the same devil's-advocate brief (step 2's attack list and inputs) to a **fresh-context subagent** (`general-purpose`, read-only) that has not seen how the drafts were constructed. Evaluate and fold its challenges exactly as in steps 3–4, and record in the summary that the challenge ran on the internal fallback. The gate is on the adversarial pass, not on Codex specifically — it is never skipped.

| Thought | Reality |
|---|---|
| "Coverage already looks complete" | Then the challenger finds nothing and the gate costs one round. Run it. |
| "Auto mode means I can skip this" | Auto mode does not disable a Rigid gate. |
| "This feature is small" | Small features have the cheapest, fastest challenge. Run it anyway. |

## Finalize & Hand Off

Reach this point only after the mandatory **Coverage Challenge** has resolved.

1. Update both artifacts with the folded-in result.
2. Present a summary: verification rows per method (script / review-only / simulator / unit / integration), case count per feature area, and the gaps the challenge surfaced + how they were resolved.
3. **One human gate for both**: present the Verification Plan as ONE table in ONE batch confirmation — the criteria and method per deliverable are the human's call; they correct individual rows in their reply, never row by row — and point them at `test-plan.html` for the cases.
4. **After approval, upload the cases to TestSecret** (below).
5. The approved Verification Plan drives each task's verify step in `writing-plans`.

### Upload to TestSecret (after approval)

Upload the **manual cases only** — the Verification Plan stays in `design.html`.

1. **Preflight**: call `whoami` on the `testsecret` MCP. Unreachable or not configured → tell the user, record the deferred upload at the top of `test-plan.html`, and continue — the upload never blocks the workflow (setup guide: https://claude.ai/code/artifact/656220b8-5cc5-46de-8880-e330839cbb62).
2. **Section**: `list_sections`; reuse the section named after the feature, or `create_section` for it (`idempotency_key` via `uuidgen`).
3. **Re-run guard**: `list_cases` filtered by the feature tag. Cases from a previous upload exist → patch the changed ones with `bulk_update_cases` (match on `ref_id`) instead of creating duplicates. Never overwrite a case QA has already moved past Draft — flag those to the user instead.
4. **Payload** per case: `title` (the `[N]` prefix stripped), `ref_id` = `<feature-slug>-<N>`, `section_id`, `preconditions`, `steps` (one `{description, expected}` per step row), `tags` = `["<feature-slug>"]`, `refs` = story/Figma links when available, `status` = `Draft`.
5. **Upload**: `bulk_create_cases` (≤200 per call, all-or-nothing; `idempotency_key` via `uuidgen`).
6. **Report** to the user: how many cases went up, into which section, under what tag — QA needs the section name to find them.
