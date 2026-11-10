---
name: write-test-plan
description: Use after the design is confirmed to write the feature's Verification Plan — the developer's own test considerations, one row per deliverable stating how it is proven done (criteria → method), appended to design.html. QA owns the manual test cases; this skill writes none and uploads nothing.
---

## When to Use

- After the feature-workflow Step 3 design gate, BEFORE `writing-plans`
- Standalone: any confirmed design whose deliverables need verification criteria

## Scope — what this skill does NOT write

QA writes the manual test cases, in TestSecret, with their own method. This skill writes **no manual cases and uploads nothing**. Its only output is the developer's Verification Plan: how dev proves each deliverable done before it reaches QA. QA's shared understanding of the feature comes from the decision summary posted to the Shortcut stories at the Step 3 gate (feature-workflow), not from this plan.

## Input

- The confirmed `design.html`: D-decisions, acceptance points, input → output examples, non-goals
- The story-analysis output when it exists. **Input validation**: check that it includes a **Behavioral Dimensions** section — the coverage challenge needs it. If missing, flag to the user and ask whether to proceed or revisit story-analysis first.

## Output

**Language: English.** Append a **"Verification Plan"** section to the feature's `design.html`, before the Execution Checklist, as one `<table>` using the shared skeleton's styling (`../feature-workflow/references/plan-doc-skeleton.html`). No separate document.

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

## Coverage Challenge (devil's advocate) — mandatory

**Type: Rigid gate. Do not skip, do not rationalize past it.** Before the Verification Plan is finalized, it MUST survive an adversarial review — by Codex when available, by a fresh-context subagent otherwise (see fallback below):

1. Write the draft Verification Plan into `design.html` first, so the challenger can read it by path. Open a Codex session with the Codex CLI: `codex exec --sandbox read-only -C <project root> -o <reply file> "<brief>"`, continuing the same thread with `codex exec resume --last`.
2. Frame Codex as a **picky devil's advocate** whose only job is to attack the plan. Send it the draft Verification Plan plus `design.html` and the story-analysis output, and ask it to find:
   - **Dimension gaps**: a Behavioral Dimension that changes behavior (account type, market, entry point, upgrade state, guest vs registered) with no row covering it
   - **Verification gaps**: a deliverable with no row, criteria that nothing can observe, a method too weak for its risk (review-only on calculation logic, a script that checks the happy path but not the failure the criteria name), a persistence flow with no integration row, a committed-fixture claim with no fixture
   - **Cost-only rows**: a Prevents that names no concrete error, a rule asserted twice, an expected value derived the way the implementation computes it, a row that would break on a harmless refactor, a presentation-math row the "Not in this section" rule already excludes
3. Evaluate each challenge with project context — accept real gaps, push back (with reasoning) on speculative ones. Iterate in the same thread (`codex exec resume --last`).
4. Fold the agreed gaps into the plan.

**Codex CLI unavailable → internal fallback, not a stop**: hand the same devil's-advocate brief (step 2's attack list and inputs) to a **fresh-context subagent** (`general-purpose`, read-only) that has not seen how the draft was constructed. Evaluate and fold its challenges exactly as in steps 3–4, and record in the summary that the challenge ran on the internal fallback. The gate is on the adversarial pass, not on Codex specifically — it is never skipped.

| Thought | Reality |
|---|---|
| "Coverage already looks complete" | Then the challenger finds nothing and the gate costs one round. Run it. |
| "Auto mode means I can skip this" | Auto mode does not disable a Rigid gate. |
| "This feature is small" | Small features have the cheapest, fastest challenge. Run it anyway. |

## Finalize & Hand Off

Reach this point only after the mandatory **Coverage Challenge** has resolved.

1. Update the Verification Plan section in `design.html` with the folded-in result
2. Present a summary: rows per method (script / review-only / simulator / unit / integration), and the gaps the challenge surfaced + how they were resolved
3. **Present the Verification Plan as ONE table in ONE batch confirmation** — the criteria and method per deliverable are the human's call; they correct individual rows in their reply — never ask about rows one at a time
4. The approved Verification Plan drives each task's verify step in `writing-plans`
