---
name: feature-workflow
description: Use when starting a new feature or when the user says "new feature", "start feature", "feature workflow" — orchestrates the full development lifecycle from story analysis through implementation to branch completion
---

# Feature Development Workflow

**Type: Rigid** — Follow this process exactly. Do not skip steps.

Orchestrates the full feature development lifecycle by invoking other skills in sequence. Each step has a human gate before proceeding.

## Prerequisites Check

Before starting, verify that required skills are available:
- `fatsecret-workflow:story-analysis`
- `fatsecret-workflow:write-test-plan`
- `fatsecret-workflow:review-task`
- `superpowers:writing-plans`
- `superpowers:test-driven-development`
- `superpowers:verification-before-completion`
- `superpowers:finishing-a-development-branch`

If superpowers skills are missing, tell the user:
> "This workflow requires the superpowers plugin. Install it with: `claude plugins add superpowers-marketplace/superpowers`"

## Workflow at a Glance

```
Step 1: Introduction        — collect input, confirm output folder
Step 2: Story Analysis      — understand stories + explore code [skip if no stories]
Step 3: Design Exploration  — architecture decisions, implementation approach
Step 4: Test Plan           — define acceptance criteria
Step 5: Implementation Plan — detailed task breakdown
Step 6: Build               — per-task loop: TDD → review → commit
Step 7: Verify & Ship       — final verification + finish branch
```

## Output Directory

All workflow artifacts for a feature are saved under a single folder:

```
docs/plans/<feature-name>/
  ├── story-analysis.md       (from Step 2)
  ├── design.md                (from Step 3)
  ├── test-plan.csv            (from Step 4)
  └── implementation-plan.md   (from Step 5)
```

The feature folder name is confirmed with the user in Step 1.

---

## Step 1 — Introduction

### 1a. Collect input

Present this to the user and collect input:

> I'll guide you through our delivery workflow:
>
> **Analyze** → understand stories, explore code for hidden impacts
> **Design** → architecture decisions and implementation approach
> **Test Plan** → define acceptance criteria
> **Plan** → detailed implementation steps
> **Build** → implement with TDD, verify against Figma, code review
>
> To get started — do you have **Shortcut Stories** (Story IDs or Iteration ID) or **Figma designs** to share?

Collect Story IDs, Iteration IDs, and Figma URLs as provided.

### 1b. Confirm output folder

> "I'll save all artifacts to `docs/plans/<feature-name>/` — does this name work?"

---

## Step 2 — Story Analysis [human gate]

**Skill**: `fatsecret-workflow:story-analysis`
**Condition**: Stories were provided (Story IDs, Iteration ID, or pasted text). If no stories, skip to Step 3.

Save output to `docs/plans/<feature-name>/story-analysis.md`.

---

## Step 3 — Design Exploration [human gate]

**Condition**: Always. Do NOT invoke any external skill for this step.

**Input varies by path:**
- **With story analysis** (Step 2 completed): Uses story-analysis output as input. Objectives: (1) Propose implementation approaches and architecture decisions for the Items (2) Identify existing code issues that affect this work (3) Design shared components and state management across Items
- **Without story analysis** (Step 2 skipped): Explore the idea from scratch — understand intent, propose approaches, present design.

Explore the codebase, discuss trade-offs, and present a design to the user. Wait for human confirmation.

Save output to `docs/plans/<feature-name>/design.md`.

---

## Step 4 — Test Plan [human gate]

**Skill**: `fatsecret-workflow:write-test-plan`
**Condition**: Always.

Uses the story-analysis output (if available) and design exploration output as input.

Save output to `docs/plans/<feature-name>/test-plan.csv`.

Wait for human confirmation before proceeding.

---

## Step 5 — Implementation Plan [human gate]

**Skill**: `superpowers:writing-plans`
**Condition**: Always.

Uses all prior outputs (story-analysis, design, test plan) as input.

**Figma design coverage is enforced here.** Every UI element in the plan must have a corresponding Figma design node. If any UI element cannot be matched to a Figma node, ask the user for the design before including it in the plan. If you must make assumptions and build UI without a design, confirm with the human first. This is the checkpoint where missing designs surface — even if Figma URLs were not provided in Step 1, any UI work in the plan requires design backing.

Save output to `docs/plans/<feature-name>/implementation-plan.md`.

Wait for human confirmation before proceeding.

---

## Step 6 — Build

**Condition**: Always. Execute each task from the implementation plan in order.

### Pre-task check

Before starting each new task, check for uncommitted changes (`git status`). If there are uncommitted changes from a previous task, commit them before proceeding.

### Per-task loop

For each task in the implementation plan, **announce the checklist before starting**:

> **Task N: [name]**
> - [ ] 6a. Implement with TDD → `superpowers:test-driven-development`
> - [ ] 6b. Review task → `fatsecret-workflow:review-task`
> - [ ] 6c. Fix issues & commit

Then execute each step in order, checking off as you go. Do NOT proceed to the next task until all three are checked.

**Step 6a — Implement with TDD**: Follow `superpowers:test-driven-development` for the current task only. Stop when the task is complete and builds.

**Step 6b — Review task**: After the task builds successfully, invoke `fatsecret-workflow:review-task` on the task's changes. This runs spec compliance, UI verification against Figma (if UI changed), and codex debate review — all while the implementation context is still fresh.

**Step 6c — Fix & commit**: Fix any issues raised by the review, then commit.

**CRITICAL: Only `review-task` marks tasks complete.** Do NOT call TaskUpdate(completed) directly. The review-task skill marks the task done after all checks pass and the human confirms. This is a structural gate — no task is "done" without review.

### Debug Verification with Utils.runTest()

When a task produces UI that cannot be triggered through normal app flow yet (e.g., a prompt method that isn't wired up), use `Utils.runTest()` to verify it:

1. Add temporary test code at the start of `Utils.runTest()` to trigger the UI element
2. Build and run the app in the simulator, trigger runTest from the debug menu
3. Take a screenshot and verify the UI (compare with Figma if applicable)
4. **Remove the test code** before proceeding to the next task

---

## Step 7 — Verify & Ship

Three sub-steps to close out the feature:

### 7a. Test plan verification

**Condition**: Always (test plan is written in Step 4).

Verify coverage against implementation. Informational — does not block.

### 7b. Verify before completion

**Skill**: `superpowers:verification-before-completion`
**Condition**: Always.

### 7c. Finish branch

**Skill**: `superpowers:finishing-a-development-branch`
**Condition**: Always.

---

## Important

- Do NOT skip steps. If a conditional step does not apply, move to the next.
- Every step with a human gate requires explicit user confirmation before proceeding.
- All artifacts save to the feature's output directory, not the skill's default location.
