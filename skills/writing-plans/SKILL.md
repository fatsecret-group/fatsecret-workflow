---
name: writing-plans
description: Use when a confirmed design needs to be turned into an executable task breakdown — produces the Execution Checklist appended to design.html, the workflow's single plan artifact, with per-task detail scaled to the work
---

# Writing Plans

## Overview

Turn a confirmed design + test plan into an executable task breakdown. The output is a lightweight **Execution Checklist** appended to `design.html` — the only plan artifact; a task that needs more carries the detail inside its own entry (see "One Artifact").

Assume the implementer is a skilled developer who knows almost nothing about this codebase — every task names its exact files, tests, and acceptance point. But do NOT front-load code: implementers read the real files at task time. Plans that embed large code blocks go stale the moment the design shifts, and their API references are unverified until something compiles. DRY. YAGNI.

**Project reality — unit + integration tests via TDD, no UI tests.** The project has one test target (`Calorie Counter Tests`) holding both layers. Tasks that own rows in the test plan's Automated Coverage follow **TDD**: write the failing test first (unit for a pure rule, integration through the real store for a flow), then implement until it passes — iterating with `xcodebuildmcp` (`build_sim` to compile, `test_sim` scoped with `-only-testing:` to the task's own test classes) until both compile cleanly and tests are green. Never run the whole target inside a task — the full suite belongs to Step 7b. UI tests and snapshot tests stay out of scope; pure-UI tasks with no testable logic keep the plain Implement step. The automated scope for a feature comes from the **Automated Coverage** section of its `test-plan.html` — tasks write those rows and nothing beyond them. All downstream verification (including re-running the task's tests) is owned by `review-task`. Plans MUST NOT prescribe UI-test-writing steps or standalone build steps.

**Commit timing.** Each task MUST end with an explicit "Commit & mark complete" step. The checklist/plan is the single source of truth for when commits happen.

**Announce at start:** "I'm using the writing-plans skill to create the execution plan."

## Codebase Knowledge (read first)

If `docs/plans/CODEBASE-KNOWLEDGE.md` exists, read it before mapping File Structure or decomposing tasks — the codebase map (entry points, state ownership, navigation, networking, UI/localization/analytics conventions, build/tooling diagnostics; task index at the top). Honor it while planning (don't restate it in the plan); consult it before exploring or when unsure how something is done here.

## Scope Check

If the spec covers multiple independent subsystems, it should have been broken into sub-project specs during story-analysis / scoping. If it wasn't, suggest breaking this into separate plans — one per subsystem. Each plan should produce working, testable software on its own.

## Confusion Protocol

Before you lock in decomposition decisions — and again whenever you're about to commit to an approach inside a task — scan for high-stakes ambiguity:

- Two plausible architectures or data models for the same requirement
- A spec detail that contradicts existing patterns and you're unsure which to follow
- A destructive or migration operation where the scope is unclear
- Missing context that would meaningfully change your approach (e.g., unknown data shape, unclear ownership boundary, unresolved design reference)

If you spot any of these: **STOP**. Do not start writing the plan. Name the ambiguity in one sentence, present 2-3 options with tradeoffs, and ask the user before proceeding. A plan built on the wrong architectural assumption wastes every task downstream — the cost of pausing to confirm is far lower than the cost of re-planning after Task 3.

This does NOT apply to routine decisions, small features, or obvious choices. Use judgment: if a reasonable engineer could read the spec and proceed without questions, proceed. The protocol is for *high-stakes* ambiguity, not every micro-decision.

## One Artifact — Checklist with Per-Task Detail

The Execution Checklist, appended to the feature's `design.html` as its final section, is the **only** plan artifact. There is no separate implementation-plan document: a second artifact restating the design and the checklist is a divergence risk, not added safety.

**Per-task detail replaces the second document.** A task's entry grows richer — in place, inside the checklist — when any of these holds; say in one line which entries got detail and why:
- **Dispatched to a subagent** → the entry must be self-contained for a fresh-context worker: interface signatures and the concrete test assertions for its TDD red step (actual XCTest; if the type under test doesn't exist yet, note the minimal stub needed so the test fails on the assertion, not on a missing symbol), within what "Code in Plans" allows.
- **Destructive migration** → the entry carries its callsite list and old-path semantics per "Migration Discipline" — these were enumerated at design time under feature-workflow's grounding rule, so reference `design.html`'s migration notes rather than re-deriving them.
- **The user asks** for more detail on a task.

(Standalone use outside `fatsecret-workflow:feature-workflow`: save a markdown plan to `docs/plans/YYYY-MM-DD-<feature-name>.md`. User preferences for location/format override all defaults.)

## File Structure

**Language: English.** The checklist or plan is executed by other engineers and agents, and its task titles become commit messages — write it in English even when the working conversation is in another language.

Before defining tasks, map out which files will be created or modified and what each one is responsible for. This is where decomposition decisions get locked in.

- Design units with clear boundaries and well-defined interfaces. Each file should have one clear responsibility.
- Prefer smaller, focused files over large ones that do too much.
- Files that change together should live together. Split by responsibility, not by technical layer.
- In existing codebases, follow established patterns. If the codebase uses large files, don't unilaterally restructure — but if a file you're modifying has grown unwieldy, including a split in the plan is reasonable.

**Folder placement — one batched confirmation.** When the plan introduces *new* files, confirm their folders with the human ONCE, as a single `AskUserQuestion`: list every new file with a proposed folder and a one-line rationale (e.g., matches a sibling file's pattern), and let the human confirm or redirect the batch. Do not guess silently — folder organization often encodes ownership and module boundaries invisible in the file tree — but do not interrupt per file either. Files being *modified* need no check.

## Code in Plans

- **Plain checklist entries: no code blocks at all.** Tasks name files, behaviors, interfaces, and test assertions.
- **Detailed entries (dispatch / migration):** exact file paths, behavior descriptions, interface signatures, and concrete test assertions. Full code blocks ONLY for new small types and critical algorithms. Everything else: describe the behavior and name the APIs to use.
- **Every API, type, or method a plan references must be verified to exist** — open the file in this session and cite the path. A signature copied from memory or an old doc is how hallucinated APIs get planned in.
- **Implementers re-read the target files at task time.** The plan is a contract for behavior, not a source snapshot; where plan and code disagree at implementation time, read the code and flag the drift.

## Migration Discipline (destructive / replacement work)

Applies whenever tasks remove or replace existing code (this is also a full-plan trigger):

1. **The source implementation is the spec.** Before writing target-side tasks, read the code being replaced and enumerate its guards, branches, inputs, and semantics; reference them in the task. Do not design the replacement from first principles.
2. **Reversible-first.** Removal tasks comment out — never hard-delete (feature-workflow Invariant 4). The actual deletion is its own task at the end of the plan, run only after human review of the commented-out state.
3. **Per-callsite confirmation.** Each removal task lists every affected callsite; the human confirms the list before the task starts. Unlisted callsites found mid-task are surfaced, not silently handled.
4. **Behavioral equivalence.** Each replacement task's Implement step ends by diffing the new path against the old: same guards, same inputs, same values (don't lose a NaN check, a default parameter, or a `shouldTrack`-style gate).

## UI Tasks: Figma Integration

If a task creates or modifies UI:

1. **Record Figma nodeIds in the task's Files entry** — one nodeId per UI file, pulled from `story-analysis` output (Source / Key UI Specs) or from the user. A UI task with no nodeId is a gap: ask the user for the design (or an explicit waiver) before finalizing the checklist/plan.
2. **The Implement step invokes `fatsecret-workflow:figma-driven-implementation`** — it queries Figma per component at implementation time and is the sole source of truth for visual values. Checklists and plans never carry pixel values.

## Execution Checklist Format

Append to `design.html` as a final section ("Execution Checklist"), using the shared skeleton's `.task-card` styling. Each task entry:

- **Title** — descriptive, referenced as "Task 3 (Target screen wiring)", never a bare "Task 3"
- **Files** — Create/Modify with exact paths; Figma nodeId for UI files
- **Tests** — which rows from the test plan's Automated Coverage this task owns (unit and/or integration, TDD), or "none (pure UI)"
- **Re-run** — the existing test classes in `Calorie Counter Tests` that exercise the code this task's Files change, named explicitly (grep the test target for the types in Files while writing the plan), or "none". This list is decided here and confirmed at the Step 5 gate; `review-task` runs exactly it plus the task's own tests and makes no selection of its own
- **Acceptance** — the observable outcome that makes this task done
- **Steps** — the standard cycle (write it once at the top of the section, not per task):
  Implement (TDD red → green via `xcodebuildmcp`, `test_sim` scoped to the task's test classes; `figma-driven-implementation` for UI; compile cleanly, fix new warnings; no simulator run — that's `review-task`'s job) → `review-task` (one pass; the caller owns the fix loop, max 3 rounds) → Knowledge Delta Check (defined below) → Commit & `TaskUpdate(completed)` (stage exactly the declared Files; staged ⊆ Files; post-commit `git status` clean; same-turn human OK per feature-workflow Invariant 1)

**Task tracking is via the harness task list** (`TaskCreate` / `TaskUpdate`) — the document is the human-readable spec; live status lives in the task list.

## Knowledge Delta Check (in every task)

After `review-task` returns `APPROVED` and before the commit: if this task's review surfaced a **human correction about how this codebase works**, or contradicted a line in `docs/plans/CODEBASE-KNOWLEDGE.md` (code wins — the file must be fixed), propose the addition/correction and ask the human — write only on an explicit yes. Otherwise skip silently. Never blocks the commit.

## No Placeholders

Every task must contain what an engineer needs to execute it. These are **plan failures** — never write them:
- "TBD", "TODO", "implement later", "fill in details"
- "Add appropriate error handling" / "add validation" / "handle edge cases"
- "Write tests for the above" (without naming the concrete assertions)
- "Similar to Task N" (state it explicitly — the engineer may read tasks out of order)
- References to types, functions, or methods that you have not verified to exist (or that no task defines)

**One exception:** external-dependency placeholders explicitly deferred during story-analysis (e.g. `// TODO: Add Avo tracking for <event> when Avo branch is merged`) are allowed in planned code — each MUST carry the dependency/event name and its unblock condition.

## Remember

- Exact file paths always; Figma nodeIds on every UI file
- One artifact — the checklist; dispatch and migration tasks carry their detail inside their own entry, and no code beyond what "Code in Plans" allows
- Verify every referenced API/type exists before the plan names it
- Commit + `TaskUpdate(completed)` only in the task's final step, after `review-task` returns APPROVED, staging exactly the declared Files
- Tasks with testable logic get TDD steps (failing test → implement, target `Calorie Counter Tests`, runs scoped with `-only-testing:`); no UI/snapshot tests, no standalone build steps, no full-suite runs
- Migration work follows Migration Discipline (comment-out first, per-callsite confirmation, behavioral equivalence)

## Self-Review

After writing the checklist/plan, look at the spec with fresh eyes and check against it:

1. **Spec coverage:** Can you point to a task for each requirement in the design + test plan? List any gaps.
2. **Placeholder scan:** Search for the "No Placeholders" patterns above. Fix them.
3. **Consistency:** Do names, signatures, and file paths match across tasks — and match the actual codebase? A function called `clearLayers()` in Task 3 but `clearFullLayers()` in Task 7 is a bug.
4. **Test mapping:** Is every row in the test plan's Automated Coverage owned by exactly one task? Does every task's Re-run list name real test classes that exist in the target?

Fix issues inline and move on. If a spec requirement has no task, add the task.

## Execution Handoff

**When invoked by `fatsecret-workflow:feature-workflow`** (the common case):
Say: *"Execution checklist appended to `design.html`"* (or *"Plan saved to `<path>`"*) — *"Ready for Step 6 — I'll execute each task's steps in order; `review-task` gates every commit."* Then return control to feature-workflow; do NOT offer alternative execution modes.

**When invoked standalone**, offer two options:
1. **Inline (recommended)** — Execute the tasks in this session: review the plan critically first and raise concerns before starting, then follow each task's steps exactly in order, stopping to ask when blocked instead of guessing.
2. **Subagent-Driven** — first make each dispatched task's entry self-contained (see "One Artifact"). Dispatch a fresh subagent per task with **only the task's Implement step** (subagents never invoke `review-task`, ask the human, or commit — feature-workflow Invariant 1); verify each result against the task's acceptance point yourself by reading the diff — never trust a bare "success" report.

In either standalone mode, `review-task` is still the only path to commit and mark tasks complete.
