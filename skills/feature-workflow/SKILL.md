---
name: feature-workflow
description: Use when starting a new feature or when the user says "new feature", "start feature", "feature workflow" — orchestrates the full development lifecycle from story analysis through implementation to branch completion
---

# Feature Development Workflow

**Type: Rigid** — Follow this process exactly. Do not skip steps.

Orchestrates the full feature development lifecycle by invoking other skills in sequence. Each step has a human gate before proceeding.

## Codebase Knowledge (read first)

Before Step 2 (Analysis), Step 3 (Design), Step 5 (Implementation Plan), and Step 6 (Build), read `docs/plans/CODEBASE-KNOWLEDGE.md` if it exists — the codebase map (entry points, state ownership, navigation, networking, UI/localization/analytics conventions, build/tooling diagnostics; task index at the top). Honor it during the relevant step (don't restate it in artifacts); consult it before exploring or when unsure how something is done here.

## Prerequisites Check

Before starting, verify that required skills are available:
- `fatsecret-workflow:story-analysis`
- `fatsecret-workflow:write-test-plan`
- `fatsecret-workflow:review-task`
- `fatsecret-workflow:writing-plans`
- `superpowers:verification-before-completion`
- `superpowers:finishing-a-development-branch`

If superpowers skills are missing, tell the user:
> "This workflow requires the superpowers plugin. Install it with: `claude plugins add superpowers-marketplace/superpowers`"

**Note on testing:** Tasks with testable logic are built with **TDD** — the engineer writes a failing unit test first, then implements until it passes, iterating with `xcodebuildmcp` (`build_sim` to compile, `test` to run). The unit-test scope for the feature is defined in the test plan's **Unit Test Coverage** section. Per-task verification — including re-running the task's unit tests — is owned by `review-task`. UI tests and snapshot tests are out of scope; the unit test target is `Calorie Counter Tests`.

## Workflow at a Glance

```
Step 1: Introduction        — collect input, confirm output folder
Step 2: Story Analysis      — understand stories + explore code [skip if no stories]
Step 3: Design Exploration  — architecture decisions, implementation approach
Step 4: Test Plan           — manual QA + unit-test coverage; mandatory Codex coverage challenge
Step 5: Implementation Plan — detailed task breakdown (plan defines each task's steps)
Step 6: Build               — execute each task per the plan; review-task is the verdict gate
Step 7: Verify & Ship       — final verification + finish branch
```

## Output Directory

All workflow artifacts for a feature are saved under a single folder:

```
docs/plans/<feature-name>/
  ├── story-analysis.html       (from Step 2)
  ├── design.html               (from Step 3)
  ├── test-plan.html            (from Step 4)
  └── implementation-plan.html  (from Step 5)
```

The feature folder name is confirmed with the user in Step 1.

### Document format — HTML with a sticky TOC

All plan artifacts (story-analysis, design, test-plan, implementation-plan) are written as **self-contained HTML** (inline CSS, no external deps) with a **left-side sticky Table of Contents**. These docs are reviewed in a browser by humans (dev / QA), not in a code editor — a sidebar TOC is far more navigable than scrolled markdown for documents this size.

Use the shared skeleton at `references/plan-doc-skeleton.html` (in this skill's directory) as the starting structure for every artifact — it carries the proven CSS and the helper classes (`.item-card`, `.task-card`, `.gate`, `.callout`, `.field-label`, `.tag`/`.items-tag`, `.meta`). The sub-skills (`story-analysis`, `write-test-plan`, `writing-plans`) describe the *content* of each doc; this skeleton supplies the *rendering*.

A project may override the format in its own CLAUDE.md (e.g. `docs/plans/<area>/CLAUDE.md` "Plan document format"). If a project explicitly asks for markdown or CSV, follow that instead.

---

## Large Projects — Phase Mode

Some efforts are too large to run as a single feature (a migration, a multi-subsystem rollout). Run them in **phase mode** instead of one giant feature-workflow pass:

1. **One project-level story-analysis, done ONCE.** Run `story-analysis` for the whole project up front, saved at the project root (`docs/plans/<project>/story-analysis.html`). It covers every phase's stories and the cross-cutting decisions — you do NOT redo it per phase.
2. **Each phase is its own feature-workflow run**, scoped to a phase subfolder:
   ```
   docs/plans/<project>/
     ├── story-analysis.html            (project-wide, done ONCE)
     ├── CLAUDE.md                      (project conventions: doc format, logging, …)
     └── phase-N-<name>/
         ├── design.html                (this phase — Step 3)
         ├── test-plan.html             (this phase — Step 4)
         └── implementation-plan.html   (this phase — Steps 5–6)
   ```
   For each phase, treat its subfolder as the feature folder and run **Steps 3–6 plus the per-phase verification (Step 7a + 7b)**; Step 7c (finish branch) is NOT per-phase — it follows point 4's timing. **Skip Step 2's story gathering** — the project-level analysis already covers it (only re-open it if a phase surfaces genuinely new stories). But Steps 3–5 STILL take the project-root `story-analysis.html` as their story-analysis input: treat it as the *"with story analysis"* path in Step 3, not *"explore from scratch."*
3. **Design decisions are numbered and shared across phases** (`D1`, `D6`, …) so later phases reference earlier ones. Record them in each phase's `design.html` and cross-link.
4. **Settle branch strategy + Step-7 timing with the human at project start.** Phases commonly accumulate on one long-lived branch with the final finish-branch (Step 7c) run once at project end rather than per phase — but confirm this rather than assuming it; don't run a per-phase merge/PR unless the human asked for one.

When the user names a phase ("Phase 5", "P5"), that phase's subfolder is the feature folder for its Step 3–6 artifacts — the project-level `story-analysis.html` + `CLAUDE.md` stay at the project root, not in the phase folder. The rest of each step applies unchanged.

---

## Step 1 — Introduction

### 1a. Collect input

Present this to the user and collect input:

> I'll guide you through our delivery workflow:
>
> **Analyze** → understand stories, explore code for hidden impacts
> **Design** → architecture decisions and implementation approach
> **Test Plan** → define acceptance criteria + unit-test coverage (Codex coverage challenge)
> **Plan** → detailed implementation steps per task
> **Build** → implement (TDD unit tests), build & run in simulator, review against Figma, commit
>
> To get started — do you have **Shortcut Stories** (Story IDs or Iteration ID) or **Figma designs** to share?

Collect Story IDs, Iteration IDs, and Figma URLs as provided.

### 1b. Scale check — offer phase mode

Before naming the output folder, judge the SIZE of what was collected. Offer **phase mode** (see "Large Projects — Phase Mode" above) ONLY when the work is a large epic / iteration that obviously spans multiple weeks and multiple releases — concretely, when **either** holds:

- the stories span **more than one iteration**, or
- there are **10+ stories**.

If neither holds, say nothing — proceed as a single feature run. Don't offer phase mode for a handful of stories in one iteration; it's overhead a small feature doesn't need.

When the threshold is met, use `AskUserQuestion` to offer the choice:
- **Single feature run** — one pass of Steps 2–7, all artifacts in one folder.
- **Phase mode** — one project-level story-analysis up front, then each phase run as its own pass (Steps 3–6 + 7a/7b).

If the user picks phase mode, record the decision + the phase breakdown in the project-root `CLAUDE.md` so every later phase run inherits it without re-asking. If the project is already set up in phases (a project-root `story-analysis.html` + `phase-*/` folders already exist), phase mode is already in effect — skip the question and proceed with the named phase.

### 1c. Confirm output folder

> "I'll save all artifacts to `docs/plans/<feature-name>/` — does this name work?"

(Phase mode: this phase's artifacts go in `docs/plans/<project>/phase-N-<name>/`; the project-level `story-analysis.html` + `CLAUDE.md` stay at the project root — confirm both.)

---

## Step 2 — Story Analysis [human gate]

**Skill**: `fatsecret-workflow:story-analysis`
**Condition**: Stories were provided (Story IDs, Iteration ID, or pasted text). If no stories, skip to Step 3.

Save output to `docs/plans/<feature-name>/story-analysis.html`.

---

## Step 3 — Design Exploration [human gate]

**Condition**: Always. Do NOT invoke any external skill for this step.

**Input varies by path:**
- **With story analysis** (Step 2 completed): Uses story-analysis output as input. Objectives: (1) Propose implementation approaches and architecture decisions for the Items (2) Identify existing code issues that affect this work (3) Design shared components and state management across Items
- **Without story analysis** (Step 2 skipped): Explore the idea from scratch — understand intent, propose approaches, present design.

Explore the codebase, discuss trade-offs, and present a design to the user. Wait for human confirmation.

Save output to `docs/plans/<feature-name>/design.html`.

---

## Step 4 — Test Plan [human gate]

**Skill**: `fatsecret-workflow:write-test-plan`
**Condition**: Always.

Uses the story-analysis output (if available) and design exploration output as input.

The test plan defines both the **manual QA acceptance cases** and a separate **Unit Test Coverage** section (the deterministic, non-UI units that get TDD unit tests in Step 6; target `Calorie Counter Tests`, no UI/snapshot tests). Before the plan is finalized, `write-test-plan` runs a **mandatory Codex devil's-advocate coverage challenge** over the whole draft (manual + unit) — this gate is not skippable.

Save output to `docs/plans/<feature-name>/test-plan.html`.

Wait for human confirmation before proceeding.

---

## Step 5 — Implementation Plan [human gate]

**Skill**: `fatsecret-workflow:writing-plans`
**Condition**: Always.

Uses all prior outputs (story-analysis, design, test plan) as input.

**Figma design coverage is enforced here.** Every UI element in the plan must have a corresponding Figma design node. If any UI element cannot be matched to a Figma node, ask the user for the design before including it in the plan. If you must make assumptions and build UI without a design, confirm with the human first. This is the checkpoint where missing designs surface — even if Figma URLs were not provided in Step 1, any UI work in the plan requires design backing.

Save output to `docs/plans/<feature-name>/implementation-plan.html`.

Wait for human confirmation before proceeding.

---

## Step 6 — Build

**Condition**: Always.

### Pre-task check

Before starting each new task, run `git status` to confirm the prior task committed cleanly. If uncommitted changes remain, investigate the root cause before starting the new task.

### Execution

Execute each task exactly as written in the plan. **The plan (defined by `fatsecret-workflow:writing-plans`) is the single source of truth for the steps inside a task.**

Invariants Step 6 enforces on top of the plan:

- `review-task` produces the `APPROVED` verdict that unlocks the task's final commit + `TaskUpdate(completed)` step.
- On `ESCALATED`, halt the task and surface the issues to the human.
- Commit + `TaskUpdate(completed)` happen exclusively in the task's final plan step.
- The review-fix loop (iterate on issues until APPROVED, capped at 3 rounds before ESCALATED) lives inside `review-task`.

---

## Step 7 — Verify & Ship

Three sub-steps to close out the feature:

### 7a. Test plan verification

**Condition**: Always (test plan is written in Step 4).

Verify coverage against implementation. Informational — does not block.

### 7b. Verify before completion

**Skill**: `superpowers:verification-before-completion`
**Condition**: Always.

Run the full unit-test suite (`xcodebuildmcp` `test`, target `Calorie Counter Tests`) as part of this verification — the per-task gates already ran each task's tests; this confirms the whole suite is green before shipping. If the target can't be built/run (e.g., a non-app worktree), surface that rather than silently skipping.

### 7c. Finish branch

**Skill**: `superpowers:finishing-a-development-branch`
**Condition**: Always.

---

## Important

- Do NOT skip steps. If a conditional step does not apply, move to the next.
- Every step with a human gate requires explicit user confirmation before proceeding.
- All artifacts save to the feature's output directory, not the skill's default location.
- **Commit policy — applies to EVERY commit/push in the workflow.** Never run `git commit` / `git push` without an explicit human OK given in the *same* turn. Authorization is **single-use and per-commit**: a "go" / "y" from an earlier turn never carries over to a later commit. `review-task`'s `APPROVED` **is** the same-turn OK for *that task's* Step 3 commit — but it authorizes only that one commit, never the next task, a push, or a housekeeping commit. Subagents NEVER commit; commits happen only in the main agent's plan Step 3 (after `review-task` returns `APPROVED`) or in Step 7, each with its own same-turn OK.
