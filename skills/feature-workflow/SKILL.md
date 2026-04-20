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
- `fatsecret-workflow:writing-plans`
- `superpowers:verification-before-completion`
- `superpowers:finishing-a-development-branch`

If superpowers skills are missing, tell the user:
> "This workflow requires the superpowers plugin. Install it with: `claude plugins add superpowers-marketplace/superpowers`"

**Note on testing:** All per-task verification is owned by `review-task`. During implementation, the engineer iterates with `xcodebuildmcp` to confirm the code compiles — that's part of implementation itself.

## Workflow at a Glance

```
Step 1: Introduction        — collect input, confirm output folder
Step 2: Story Analysis      — understand stories + explore code [skip if no stories]
Step 3: Design Exploration  — architecture decisions, implementation approach
Step 4: Test Plan           — define acceptance criteria
Step 5: Implementation Plan — detailed task breakdown (plan defines each task's steps)
Step 6: Build               — execute each task per the plan; review-task is the verdict gate
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
> **Plan** → detailed implementation steps per task
> **Build** → implement, build & run in simulator, review against Figma, commit
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

**Skill**: `fatsecret-workflow:writing-plans`
**Condition**: Always.

Uses all prior outputs (story-analysis, design, test plan) as input.

**Figma design coverage is enforced here.** Every UI element in the plan must have a corresponding Figma design node. If any UI element cannot be matched to a Figma node, ask the user for the design before including it in the plan. If you must make assumptions and build UI without a design, confirm with the human first. This is the checkpoint where missing designs surface — even if Figma URLs were not provided in Step 1, any UI work in the plan requires design backing.

Save output to `docs/plans/<feature-name>/implementation-plan.md`.

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

### 7c. Finish branch

**Skill**: `superpowers:finishing-a-development-branch`
**Condition**: Always.

---

## Important

- Do NOT skip steps. If a conditional step does not apply, move to the next.
- Every step with a human gate requires explicit user confirmation before proceeding.
- All artifacts save to the feature's output directory, not the skill's default location.
