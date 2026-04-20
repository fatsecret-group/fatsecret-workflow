---
name: writing-plans
description: Use when you have a spec or requirements for a multi-step task, before touching code
---

# Writing Plans

## Overview

Write comprehensive implementation plans assuming the engineer has zero context for our codebase and questionable taste. Document everything they need to know: which files to touch for each task, complete code, exact build/verify commands. Give them the whole plan as bite-sized tasks. DRY. YAGNI.

Assume they are a skilled developer, but know almost nothing about our toolset or problem domain.

**Project reality — no automated tests.** Our project currently has no unit test or UI test infrastructure. During implementation, the engineer iterates with `xcodebuildmcp` (`build_sim`) until the code compiles cleanly. All downstream verification is owned by `review-task`. Plans MUST NOT prescribe write-failing-test steps, UI-test-writing steps, or standalone build steps.

**Commit timing.** Each task MUST end with an explicit "Commit & mark complete" step. The plan is the single source of truth for when commits happen.

**Announce at start:** "I'm using the writing-plans skill to create the implementation plan."

**Save plans to:** `docs/plans/<feature-name>/implementation-plan.md` when invoked by `fatsecret-workflow:feature-workflow`. Otherwise `docs/superpowers/plans/YYYY-MM-DD-<feature-name>.md`.
- (User preferences for plan location override these defaults)

## Scope Check

If the spec covers multiple independent subsystems, it should have been broken into sub-project specs during brainstorming. If it wasn't, suggest breaking this into separate plans — one per subsystem. Each plan should produce working, testable software on its own.

## Confusion Protocol

Before you lock in decomposition decisions in the next section — and again whenever you're about to commit to an approach inside a task — scan for high-stakes ambiguity:

- Two plausible architectures or data models for the same requirement
- A spec detail that contradicts existing patterns and you're unsure which to follow
- A destructive or migration operation where the scope is unclear
- Missing context that would meaningfully change your approach (e.g., unknown data shape, unclear ownership boundary, unresolved design reference)

If you spot any of these: **STOP**. Do not start writing the plan. Name the ambiguity in one sentence, present 2-3 options with tradeoffs, and ask the user before proceeding. A plan built on the wrong architectural assumption wastes every task downstream — the cost of pausing to confirm is far lower than the cost of re-planning after Task 3.

This does NOT apply to routine decisions, small features, or obvious choices. Use judgment: if a reasonable engineer could read the spec and proceed without questions, proceed. The protocol is for *high-stakes* ambiguity, not every micro-decision.

Once resolved (or confirmed absent), continue to File Structure.

## File Structure

Before defining tasks, map out which files will be created or modified and what each one is responsible for. This is where decomposition decisions get locked in.

- Design units with clear boundaries and well-defined interfaces. Each file should have one clear responsibility.
- You reason best about code you can hold in context at once, and your edits are more reliable when files are focused. Prefer smaller, focused files over large ones that do too much.
- Files that change together should live together. Split by responsibility, not by technical layer.
- In existing codebases, follow established patterns. If the codebase uses large files, don't unilaterally restructure - but if a file you're modifying has grown unwieldy, including a split in the plan is reasonable.

This structure informs the task decomposition. Each task should produce self-contained changes that make sense independently.

**Folder placement — confirm with human, no skipping.** Whenever the plan introduces *new* files, you MUST ask the human which folder each new file belongs in before writing the File Structure section. List the proposed new files and, for each, either:
- propose a specific folder with a one-line rationale (e.g., matches a sibling file's pattern), and ask the human to confirm or redirect, or
- ask outright if no obvious home exists.

Do not guess folder placement from codebase pattern-matching alone — folder organization often encodes ownership, module boundaries, or conventions that aren't visible in the file tree. Wait for the human's answer before proceeding. This rule has no exceptions, even for "obvious" placements.

Files being *modified* don't need this check — their location is already fixed.

## UI Tasks: Figma Integration

If a task creates or modifies UI, the plan MUST wire in `fatsecret-workflow:figma-driven-implementation` so implementation values come from Figma per-component rather than only from the plan's code block. Why: you extract Figma values once while planning (error-prone for multi-component screens), but `figma-driven-implementation` queries per component at implementation time — it catches drift before `review-task` has to spot it visually.

Required for every UI task:

1. **Record Figma nodeIds in the Files section.** For each UI file created or modified, list the component's Figma `nodeId` next to the path — pull these from `story-analysis` output (Source / Key UI Specs) or from the user. Example:
   ```
   **Files:**
   - Create: `Views/TextInputField.swift` — Figma node `123:456`
   - Modify: `Views/FoodDiary.swift:88-120` — Figma node `789:012`
   ```

2. **Instruct the implementer to invoke `figma-driven-implementation` in Step 1.** The skill queries Figma per component and extracts exact values (dimensions, padding, font, color, corner radius). It is the source of truth for visual values — not the plan's code block.

3. **Treat the plan's code block as a starting point, not a contract for pixel values.** Write your best extraction of Figma values into the code block, but expect the implementer to reconcile each component against Figma. If you're unsure about a specific value, mark it `// TODO: verify against Figma node <nodeId>` rather than guessing.

If a UI task has no corresponding Figma node, stop and surface this to the user — per `feature-workflow` Step 5, UI work without design backing needs explicit human confirmation before the plan proceeds.

## Bite-Sized Task Granularity

**Each step is one action (a few minutes, except Implement which can be longer):**
- "Implement the component / modify the file" - step (includes iterating with `xcodebuildmcp` until it compiles)
- "Invoke review-task" - step (produces verdict only)
- "Commit & mark complete" - step (runs after APPROVED verdict)

## Plan Document Header

**Every plan MUST start with this header:**

```markdown
# [Feature Name] Implementation Plan

> **For agentic workers:** Execute task-by-task via `fatsecret-workflow:feature-workflow` Step 6. Each task ends with an explicit Step 3 that commits AND calls `TaskUpdate(completed)`. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** [One sentence describing what this builds]

**Architecture:** [2-3 sentences about approach]

**Tech Stack:** [Key technologies/libraries]

---
```

## Task Structure

Each task uses this three-step shape.

````markdown
### Task N: [Component Name]

**Files:**
- Create: `exact/path/to/File.swift` — Figma node `<nodeId>` (UI files only)
- Modify: `exact/path/to/Existing.swift:123-145` — Figma node `<nodeId>` (UI files only)

- [ ] **Step 1: Implement**

```swift
// Complete code for the file or modification — no TBDs, no "similar to above".
struct SomeView: View {
    var body: some View {
        Text("Hello")
    }
}
```

**For UI tasks:** Before writing or adjusting visual values in the code block above, invoke `fatsecret-workflow:figma-driven-implementation` with the nodeIds listed in the Files section. That skill queries Figma per component and gives you the authoritative values — reconcile the code block against its output before compiling. See the "UI Tasks: Figma Integration" section for the rationale.

During implementation, iterate with `xcodebuildmcp` (`build_sim` on the session's default scheme/simulator) to confirm the code compiles cleanly. Fix any compile errors or new warnings before moving on. No simulator run, no UI inspection, no test scaffolding — all of that is `review-task`'s job in Step 2.

- [ ] **Step 2: Review task**

Invoke `fatsecret-workflow:review-task` on this task's changes. It returns `APPROVED` or `ESCALATED`. Proceed to Step 3 only on `APPROVED`. On `ESCALATED`, surface the issues to the human and wait for instructions.

- [ ] **Step 3: Commit & mark complete**

Only reachable when Step 2 returned `APPROVED`.

```bash
git add <exact paths changed in this task>
git commit -m "<conventional commit message describing this task>"
```

Then call `TaskUpdate(taskId=<this task's id>, status=completed)`.

This is the ONLY point in the plan where commit + `TaskUpdate(completed)` happen for this task.
````

## No Placeholders

Every step must contain the actual content an engineer needs. These are **plan failures** — never write them:
- "TBD", "TODO", "implement later", "fill in details"
- "Add appropriate error handling" / "add validation" / "handle edge cases"
- "Write tests for the above" (without actual test code)
- "Similar to Task N" (repeat the code — the engineer may be reading tasks out of order)
- Steps that describe what to do without showing how (code blocks required for code steps)
- References to types, functions, or methods not defined in any task

## Remember
- Exact file paths always
- Complete code in every step — if a step changes code, show the code
- Exact commands with expected output
- DRY, YAGNI
- Commit + `TaskUpdate(completed)` only in the task's Step 3, after `review-task` returns APPROVED
- No test-writing steps — project has no unit test or UI test infrastructure
- No standalone Build step — compile verification is part of Step 1 Implement
- UI tasks list Figma nodeIds in the Files section and invoke `figma-driven-implementation` in Step 1

## Self-Review

After writing the complete plan, look at the spec with fresh eyes and check the plan against it. This is a checklist you run yourself — not a subagent dispatch.

**1. Spec coverage:** Skim each section/requirement in the spec. Can you point to a task that implements it? List any gaps.

**2. Placeholder scan:** Search your plan for red flags — any of the patterns from the "No Placeholders" section above. Fix them.

**3. Type consistency:** Do the types, method signatures, and property names you used in later tasks match what you defined in earlier tasks? A function called `clearLayers()` in Task 3 but `clearFullLayers()` in Task 7 is a bug.

If you find issues, fix them inline. No need to re-review — just fix and move on. If you find a spec requirement with no task, add the task.

## Execution Handoff

After saving the plan:

**When invoked by `fatsecret-workflow:feature-workflow`** (the common case):
Say: *"Plan saved to `<path>`. Ready for Step 6 — I'll execute each task's steps in order, and `review-task` will gate commit + completion."* Then return control to feature-workflow; do NOT offer alternative execution modes.

**When invoked standalone** (no feature-workflow orchestration):
Offer two options:
1. **Inline (recommended here)** — Execute tasks in this session using `superpowers:executing-plans`.
2. **Subagent-Driven** — Dispatch a fresh subagent per task using `superpowers:subagent-driven-development`.

In either standalone mode, `review-task` is still the only path to commit and mark tasks complete.
