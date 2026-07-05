---
name: review-task
description: Use after implementing a task — one review pass covering spec compliance, UI verification against Figma (including regression on existing elements), Codex debate, unit tests, and simplification/style. Returns ISSUES for the caller's fix loop, or runs the mandatory human gate and returns APPROVED / ESCALATED. Does NOT commit or mark the task complete; the caller does that in the task's commit step.
---

## When to Use

- After an implementation task compiles cleanly (Implement step finished), before the task's commit step
- Re-invoked by the caller after fixing issues from a previous pass — **the caller owns the review-fix loop** (max 3 rounds, then the caller treats the task as `ESCALATED`)

## Contract

This skill performs **one review pass per invocation**:
- Reviews spec compliance, Figma UI match + existing-element regression, Codex debate, and simplification/style
- Runs the task's unit tests (target `Calorie Counter Tests`)
- Builds the app and runs the simulator for UI verification
- If the changed UI cannot be reached via normal app flow, autonomously adds a temporary trigger at the start of `Utils.runTest()`, exercises it from the debug menu, screenshots, then removes the trigger before returning
- Any check that cannot run is reported as **UNVERIFIED — it never silently passes** (exception: an unavailable Codex MCP is stop-and-ask, not UNVERIFIED — see check 3)
- Returns one of:
  - **`ISSUES`** + the agreed issue list → the caller fixes (fixes only, no commit) and re-invokes this skill. No human gate on this path.
  - **`APPROVED` / `ESCALATED`** — only after the mandatory human gate, when the pass is clean.
- **The human gate is mandatory before any `APPROVED`, in every mode, including auto.** Auto mode reduces LLM-initiated questions, but this gate is a harness block, not a courtesy. Do not rationalize past it.

Commit + task completion are the caller's responsibility, executed in the task's commit step only when this skill returns `APPROVED`.

## Process

Dispatch a single reviewer subagent with the following instructions. The reviewer handles all five checks in one pass. It does NOT fix code — it only identifies and reports issues.

### Reviewer Subagent Instructions

The reviewer performs these checks in order:

**1. Spec Compliance**
- Read the task description from the execution checklist / implementation plan — or, on the Lite Track, from the harness task (its declared files, acceptance point, unit-test scope, and Figma nodeId/waiver)
- Read the changed files (use git diff)
- Verify all requirements in the task are implemented — nothing missing, **nothing extra**: any change the task didn't ask for (adjacent "improvements", edits to working code, files outside the task's declared list) is an issue, not a bonus

**2. UI Verification (only if UI was changed)**
- Build the app using XcodeBuildMCP
- Launch in the simulator and navigate to the affected screen via normal app flow
- If normal flow cannot reach the changed UI yet (e.g., a prompt method not wired up), add a temporary trigger at the start of `Utils.runTest()` that drives the UI element, rebuild, and invoke runTest from the debug menu
- Take a screenshot of the affected screen
- Fetch the corresponding Figma design screenshot using Figma MCP
- Compare implementation against design — layout, colors, spacing, text, icons
- **Regression check on existing elements:** the diff must not have removed or shifted anything on that screen the task didn't intend — verify pre-existing elements (badges, ticks, rows, decorations) are still present and correct, against the Figma design and/or the screen's prior state
- **Upgrade-path caveat:** a fresh simulator install cannot exercise upgrade-from-older-build state; if the task touches version-gated or upgrade-path logic, mark that path **UNVERIFIED** in the report — do not claim it passed
- **Before returning the report, remove any `Utils.runTest()` trigger you added and rebuild to confirm it still compiles.**
- Skip this check only if the task has no UI changes

**3. Codex Debate Review**

The reviewer debates with Codex directly to reach consensus on what issues exist. The reviewer only identifies issues; it does not fix code.

**Step 3a: Send to Codex**
- Use `mcp__codex__codex` to send the diff and task intent
- Ask Codex to review for: bugs, logic errors, edge cases, missing error handling, style issues
- If `mcp__codex__codex` is unavailable, stop and ask the user to enable it — do not skip the debate and do not downgrade it to UNVERIFIED

**Step 3b: Evaluate each issue Codex raises**
- **Agree**: If the issue is valid, record it as a confirmed issue
- **Disagree**: If the issue is not valid (false positive, project convention, etc.), explain reasoning to Codex using `mcp__codex__codex-reply`

**Step 3c: Iterate until consensus**
- Use `mcp__codex__codex-reply` to continue the conversation — keep the thread, don't start new sessions
- Debate until both sides agree on which issues are real
- Push back on over-engineered suggestions — the right fix is the minimal correct fix

**4. Unit Tests**
- Run the task's unit tests with XcodeBuildMCP (`test`) against the `Calorie Counter Tests` target
- The unit tests this task added/changed — and any existing tests the change could affect — must pass. A red test is a confirmed issue
- If the test target cannot be built/run (e.g., a worktree without the Xcode project), report this check as **UNVERIFIED** — never as passed

**5. Simplification & Style**

Simplicity is this project's #1 review principle (global CLAUDE.md Core Principles). Scan the diff itself — not just for bugs, but for bloat:

- **Collapsible constructs**: unnecessary new files/types/functions/tokens/wrappers; a counter or flag an existing notification/protocol already provides; type-checks or state mutations in callers that belong on the type itself (Tell, Don't Ask); knowledge placed on the wrong type
- **Duplication**: a helper/constant/parser that already exists elsewhere in the codebase (grep before accepting a new one)
- **Comment economy**: comments narrating what the code says, fix-context comments, multi-line comments patching unclear code (the code should be fixed instead)
- **Formatting**: edits to untouched empty lines or unrelated whitespace
- **Naming & responsibility**: does the name say what the function does (a `fetch()` that also writes back is a `sync()`)? one clear responsibility per function/file?

Each finding here is an issue like any other — it enters the same report and fix loop.

### Output Format

The reviewer compiles all findings into a single report:

```
## Post-Task Review: [Task Name]

### 1. Spec Compliance: PASS / ISSUES
### 2. UI Verification: PASS / ISSUES / UNVERIFIED / N/A
### 3. Codex Review: PASS / ISSUES
### 4. Unit Tests: PASS / ISSUES / UNVERIFIED
### 5. Simplification & Style: PASS / ISSUES
[Details per check — every PASS must cite the evidence that backs it: the exact command / tool call and its result (test output, screenshot path, Figma node compared), and every UNVERIFIED must state why it could not run. A PASS with no evidence is not a PASS.]

### Unverified
[Explicit list of everything that could not be exercised — empty only if all checks ran]

### Recommendation: APPROVE / ISSUES
[Summary. This is the reviewer's INPUT to the caller/human gate — never the skill's final verdict.]
```

If the task has **no Figma reference AND no unit-testable logic** (e.g. backend-integration work), the Recommendation must say so explicitly: "effectively unverified — static review only" — so the human gate reflects reality. (`feature-workflow` Step 7a requires real-backend verification or an explicit waiver for such features before shipping.)

### After the reviewer returns

- **Any check reports ISSUES** → return **`ISSUES`** + the agreed issue list to the caller. Do not run the human gate. The caller fixes (no commit) and re-invokes this skill; the caller tracks rounds (max 3).
- **Clean pass** (Recommendation APPROVE — possibly with UNVERIFIED items) → proceed to the human gate below.

### Verdict and Handoff (mandatory human gate)

**The skill's verdict is set by the human, not by the reviewer.**

1. Post a concise summary to the user: what was implemented, what was reviewed, fixes made across rounds, the reviewer's recommendation.
2. **Show the human what they are approving:** the diff summary — files changed plus the key hunks (for larger diffs, the exact `git diff`/`git show` command to view it) — and, verbatim, every item from the report's **Unverified** list. An Approve must be informed consent; approving with UNVERIFIED items is an explicit waiver and is recorded as such.
3. **Call `AskUserQuestion`** with:
   - `question`: "Review for `<task name>` complete. Recommendation: APPROVE. Unverified: `<list or none>`. Approve commit + TaskUpdate?"
   - `header`: "Task review"
   - `options`: `"Approve"` (proceed to commit and TaskUpdate) / `"Escalate"` (surface issues, block commit, wait for instructions)
   - `multiSelect`: false
4. Map the user's answer to the verdict:
   - `Approve` → return `APPROVED` to the caller. Caller proceeds to the task's commit step.
   - `Escalate`, "Other" with a non-approve response, or any message that isn't unambiguous approval → return `ESCALATED`. Caller stops and waits for instructions.

**You MUST call `AskUserQuestion` before returning `APPROVED`.** The call is the gate. A reviewer subagent saying "APPROVE" is not a verdict — it is a recommendation that feeds into the human gate. Do NOT infer, assume, or synthesize the human's answer from prior messages, auto mode, or the reviewer's recommendation.

## Important

- A user's `Approve` is a **single-use authorization for THIS task's commit only** — it does not pre-authorize the next task, a later push, or any housekeeping commit. Each commit needs its own same-turn OK. Subagents NEVER commit; the main agent runs the commit in the task's commit step after this skill returns `APPROVED`.
- The reviewer subagent only reports issues; the caller fixes; this skill only produces `ISSUES` or a human-gated verdict
- UI verification requires a Figma URL — check the execution checklist / implementation plan or story-analysis output for the relevant Figma node
- For the Codex debate: respond to every issue Codex raises, be willing to debate or concede, keep it simple

## Red Flags — stop and correct course

| Thought | Reality |
|---|---|
| "Reviewer said APPROVE, I can commit" | Reviewer's output is a RECOMMENDATION. The gate is `AskUserQuestion`. |
| "Auto mode means I don't need to ask" | Auto mode does not disable Rigid-skill gates. |
| "User said 'go' earlier, that covers this" | `go` authorizes executing the plan, not skipping a per-task human gate. |
| "This task was trivial, confirmation would be noise" | Then the user will click Approve in one tap. Call it anyway. |
| "I'll batch gates at the end of the plan" | Every task's gate is independent. Don't batch. |
| "Build/tests can't run here, mark it SKIPPED and move on" | Report it UNVERIFIED and surface it verbatim at the human gate. Unverified ≠ passed. |
| "The diff is too big to show, they trust me" | Show the file list + key hunks, or give the exact command to view it. Approval must be informed. |
