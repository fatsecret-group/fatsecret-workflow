---
name: writing-plans
description: Use when you have a spec or requirements for a multi-step task, before touching code
---

# Writing Plans

## Overview

Write comprehensive implementation plans assuming the engineer has zero context for our codebase and questionable taste. Document everything they need to know: which files to touch for each task, complete code, exact build/verify commands. Give them the whole plan as bite-sized tasks. DRY. YAGNI.

Assume they are a skilled developer, but know almost nothing about our toolset or problem domain.

**Project reality — unit tests via TDD, no UI tests.** The project has a unit test target (`Calorie Counter Tests`). Tasks that touch **testable logic** (pure functions, view models, parsers, business rules, state transitions) follow **TDD**: write a failing unit test first, then implement until it passes — iterating with `xcodebuildmcp` (`build_sim` to compile, `test` to run) until both compile cleanly and tests are green. UI tests and snapshot tests stay out of scope; pure-UI tasks with no testable logic keep the plain Implement step. The unit-test scope for a feature comes from the **Unit Test Coverage** section of its `test-plan.html`. All downstream verification (including re-running the task's tests) is owned by `review-task`. Plans MUST NOT prescribe UI-test-writing steps or standalone build steps.

**Commit timing.** Each task MUST end with an explicit "Commit & mark complete" step. The plan is the single source of truth for when commits happen.

**Announce at start:** "I'm using the writing-plans skill to create the implementation plan."

**Save plans to:** `docs/plans/<feature-name>/implementation-plan.html` when invoked by `fatsecret-workflow:feature-workflow` (self-contained HTML — see "Plan Document Format" below). Otherwise `docs/superpowers/plans/YYYY-MM-DD-<feature-name>.md` (markdown — the superpowers standalone default).
- (User preferences for plan location/format override these defaults)

## Codebase Knowledge (read first)

If `docs/plans/CODEBASE-KNOWLEDGE.md` exists, read it before mapping File Structure or decomposing tasks — the codebase map (entry points, state ownership, navigation, networking, UI/localization/analytics conventions, build/tooling diagnostics; task index at the top). Honor it while planning (don't restate it in the plan); consult it before exploring or when unsure how something is done here. This is the read-side counterpart to Step 2.5 ("Capture codebase knowledge"). When invoked via `fatsecret-workflow:feature-workflow`, this overlaps with that skill's "read first" before Step 5 — reading again here is harmless.

## Scope Check

If the spec covers multiple independent subsystems, it should have been broken into sub-project specs during story-analysis / scoping. If it wasn't, suggest breaking this into separate plans — one per subsystem. Each plan should produce working, testable software on its own.

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
- **Tasks with testable logic — TDD:** "Write the failing unit test(s)" - step (the assertions from the test plan's Unit Test Coverage, in the `Calorie Counter Tests` target), then "Implement until the test passes" - step (iterate with `xcodebuildmcp`: `build_sim` to compile, `test` to run, until green)
- **Pure-UI / no-logic tasks:** "Implement the component / modify the file" - step (includes iterating with `xcodebuildmcp` until it compiles)
- "Invoke review-task" - step (produces verdict only)
- "Capture codebase knowledge" - step (after APPROVED, before commit; conditional — see Task Structure Step 2.5)
- "Commit & mark complete" - step (runs after APPROVED verdict)

## Plan Document Format

When invoked by `fatsecret-workflow:feature-workflow`, write the plan as **self-contained HTML with a left-side sticky TOC** — copy the structure from the shared skeleton at `../feature-workflow/references/plan-doc-skeleton.html` (in the `fatsecret-workflow` plugin). Render each task as a `.task-card`, its steps as an `<ol>` / labelled `<p>`s, code as `<pre><code>`, and the commit gate as a `.gate` div. The `<nav id="toc">` lists every task (anchor → the task's `id`). (Standalone superpowers plans stay markdown — see "Save plans to" above.)

**Task tracking is via the harness task list** (`TaskCreate` / `TaskUpdate`), NOT document checkboxes — the HTML plan is the human-readable spec; live status lives in the task list.

## Plan Document Header

**Every plan MUST open with this header** — `<h1>` + a `p.meta` line + a `.callout` for agentic workers:

```html
<h1>[Feature Name] Implementation Plan</h1>

<p class="meta">
  <strong>Goal</strong>: [one sentence describing what this builds] &nbsp;|&nbsp;
  <strong>Architecture</strong>: [2-3 sentences about approach] &nbsp;|&nbsp;
  <strong>Tech Stack</strong>: [key technologies/libraries] &nbsp;|&nbsp;
  <strong>Inputs</strong>: <a href="story-analysis.html">story-analysis.html</a>, <a href="design.html">design.html</a>, <a href="test-plan.html">test-plan.html</a>
</p>

<div class="callout">
  <strong>For agentic workers:</strong> Execute task-by-task via <code>fatsecret-workflow:feature-workflow</code> Step 6.
  Each task ends with an explicit Step 3 that commits AND calls <code>TaskUpdate(completed)</code>.
</div>
```

## Task Structure

Each task is a `.task-card` with the same shape (Implement → Review → Capture knowledge → Commit). Give the card a `id` so the TOC can link to it. When you reference a task anywhere (TOC, cross-links, commit message), use its descriptive title alongside the code — e.g. "Task 3 (Target screen wiring)", never a bare "Task 3".

```html
<div class="task-card">
<h3 id="t-n"><span class="task-id">Task N</span> — [Component Name]</h3>

<p><span class="field-label">Files:</span>
  Create <code>exact/path/to/File.swift</code> — Figma node <code>&lt;nodeId&gt;</code> (UI files only);
  Modify <code>exact/path/to/Existing.swift:123-145</code> — Figma node <code>&lt;nodeId&gt;</code> (UI files only)
</p>

<p><span class="field-label">Step 1 — Implement:</span></p>
<p><strong>TDD (tasks with testable logic):</strong> write the failing unit test first (Step 1a), then the implementation (Step 1b). Run the test with <code>xcodebuildmcp</code> (<code>test</code>) and confirm it fails for the right reason before implementing. If the type/API under test doesn't exist yet, add the minimal stub (empty type or method signature) so the test <em>compiles and fails on the assertion</em> — a missing-symbol compile error is not a valid red. Pure-UI / no-logic tasks skip straight to the implementation block.</p>
<pre><code>// Step 1a — failing unit test (Calorie Counter Tests target). Actual XCTest, no placeholders.
final class SomeViewModelTests: XCTestCase {
    func test_total_sumsEntries() {
        XCTAssertEqual(SomeViewModel(entries: [1, 2]).total, 3)
    }
}
</code></pre>
<pre><code>// Step 1b — implementation until the test passes — no TBDs, no "similar to above".
struct SomeView: View {
    var body: some View {
        Text("Hello")
    }
}
</code></pre>
<p><strong>For UI tasks:</strong> before writing or adjusting visual values above, invoke <code>fatsecret-workflow:figma-driven-implementation</code> with the nodeIds in the Files row — it queries Figma per component and is the source of truth for visual values; reconcile the code block against its output before compiling (see "UI Tasks: Figma Integration"). During implementation, iterate with <code>xcodebuildmcp</code> (<code>build_sim</code> on the session's default scheme/simulator to compile, and for TDD tasks <code>test</code> until the unit tests are green) until it compiles cleanly; fix any new warnings. No simulator run, no UI inspection — that is <code>review-task</code>'s job in Step 2.</p>

<p><span class="field-label">Step 2 — Review:</span> Invoke <code>fatsecret-workflow:review-task</code> on this task's changes. It returns <code>APPROVED</code> or <code>ESCALATED</code>. Proceed to Step 2.5 only on <code>APPROVED</code>; on <code>ESCALATED</code>, surface the issues to the human and wait for instructions.</p>

<p><span class="field-label">Step 2.5 — Capture codebase knowledge:</span> After <code>review-task</code> returns <code>APPROVED</code> and before committing, look back over this task's code-review conversation — the Codex debate, the review-fix loop, and the human gate — and list every <em>correction the human made</em>: each place the reviewer, Codex, or your first attempt was wrong about how this codebase actually works. Filter that list through the <code>docs/plans/CODEBASE-KNOWLEDGE.md</code> "Trust Contract" inclusion test — keep ONLY which-of-many-is-canonical (disambiguation), non-local invariants, "don't do X" traps, runtime/timing behaviour, and intent→entry routing; drop anything <code>grep</code> gives in one shot (signatures, file/dir locations), module internals, behavioural rules (those live in <code>CLAUDE.md</code>), and one-off fixes (git log owns those). <strong>In the same pass, reconcile against existing entries:</strong> check whether anything you learned this task — the code's actual behaviour, or a human correction — <em>contradicts</em> a line already in <code>CODEBASE-KNOWLEDGE.md</code> (stale / now-wrong / superseded). Per the file's own Trust Contract ("If the code disagrees, the code wins — fix this file"), raise every such conflict to the human and propose the fix or removal. If anything passes the filter (additions) <strong>or any conflict was found</strong>, show the proposed entries and conflicts and <strong>ask the human</strong> before touching <code>docs/plans/CODEBASE-KNOWLEDGE.md</code> — add, correct, or delete only on an explicit yes. If nothing surfaced, nothing passes the filter, and no conflict exists, skip silently. This step never blocks the commit.</p>

<div class="gate">
Step 3 — Commit &amp; mark complete (only reachable when Step 2 returned <code>APPROVED</code>):
<code>git add &lt;exact paths changed in this task&gt;</code> → <code>git commit -m "&lt;conventional commit message&gt;"</code> → then <code>TaskUpdate(taskId=&lt;this task's id&gt;, status=completed)</code>.
This is the ONLY point in the plan where commit + <code>TaskUpdate(completed)</code> happen for this task.
</div>
</div>
```

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
- Between Review and Commit, run the "Capture codebase knowledge" step (Step 2.5): list the human's code-review corrections, filter by `CODEBASE-KNOWLEDGE.md`'s Trust Contract, reconcile against existing entries (raise any line the latest context contradicts — code wins), and ask before recording/correcting — never auto-write, never block the commit
- Tasks with testable logic include a TDD unit-test step (Step 1a failing test → Step 1b implement, target `Calorie Counter Tests`); UI tests and snapshot tests stay out of scope
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
