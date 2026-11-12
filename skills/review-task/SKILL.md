---
name: review-task
description: Use after implementing a task — one review pass covering a review triage (what the human must read, and why), spec compliance, UI verification against Figma (including regression on existing elements), Codex debate, the task's declared verification, and simplification/style. Returns ISSUES for the caller's fix loop, or runs the mandatory human gate and returns APPROVED / ESCALATED. Does NOT commit or mark the task complete; the caller does that in the task's commit step.
---

## When to Use

- After an implementation task compiles cleanly (Implement step finished), before the task's commit step
- Re-invoked by the caller after fixing issues from a previous pass — **the caller owns the review-fix loop** (max 3 rounds, then the caller treats the task as `ESCALATED`)

## Contract

This skill performs **one review pass per invocation**:
- Reviews spec compliance, Figma UI match + existing-element regression, Codex debate, and simplification/style
- Runs exactly the task's Verification rows and Re-run list (scoped tests, committed scripts)
- Builds the app and runs the simulator for UI verification
- If the changed UI cannot be reached via normal app flow, autonomously adds a temporary trigger at the start of `Utils.runTest()`, exercises it from the debug menu, screenshots, then removes the trigger before returning
- Any check that cannot run is reported as **UNVERIFIED — it never silently passes** (exception: when Codex cannot be reached at all, check 3 switches to the built-in `/code-review` fallback — the adversarial pass still runs, so it is neither UNVERIFIED nor skipped)
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
- Read the task description from the execution checklist / implementation plan — or, on the Lite Track, from the harness task (its declared files, acceptance point, verification, and Figma nodeId/waiver)
- Read the changed files (use git diff)
- **Review triage — what the human must read.** Sort the diff so the human's attention goes where a mistake is costly and invisible, and skips what a glance, the compiler or a screen run already covers. Triage finds no bugs — checks 2–5 do that.
  1. **Intent and expected scope**: the task's acceptance point is the intent; its declared Files, plus the tests and fixtures that exercise them, are the expected scope.
  2. **`REVIEW.md`**: look for it from each changed file's directory up to the git root — it is often git-excluded and local-only, so search the disk, never git history, and in a git worktree that lacks it read the main checkout's copy. Take its "Sensitive areas" table and each area's "Reviewer asks"; the table replaces the default list below. An area is touched when the diff changes it, not when changed code merely reads it. Use "Reviewer asks" as the starting point and ask what fits this change.
  3. **Layer the change**: group the changed lines into layers by what they do, named in this codebase's terms ("Diary cell layout", "check-in trigger rule"). One hunk can split across layers; a layer can span files. Every changed line sits in exactly one layer.
  4. **Tier each layer** by what the hunk does, never by where it lives — a hunk in a view file that changes navigation, gating, persistence or an analytics event is logic:
     - **Must-review** — a mistake costs users or money and no test or screen run would show it: a sensitive area (`REVIEW.md`'s table, or by default local persistence and schema migration; upgrade and version-gated paths; user data and preference keys; auth, entitlements, payments; shared API and analytics contracts — payload shape, field and event names; models shared across modules; build, signing, release and runtime config — flags, timeouts, limits; secrets, PII and permissions); removed behaviour, data or error handling; concurrency, ordering, retry and fallback logic; a hunk outside expected scope that changes behaviour, or expected scope the change left undone.
     - **Spot-check** — ordinary logic: business rules, calculations, internal refactors, a new path behind a flag; also any test hunk that deletes or weakens an assertion.
     - **Skip** — layout and styling, user-visible copy, docs and comments, test code, formatting, renames the compiler checks.
  5. **For each must-review layer**: read its callers before stating the impact — a claim inferred only from references is a guess. Then find what guards it: the tests and verification scripts that actually exercise the changed lines (one that only touches the file does not count). Write the guard in plain words a non-author understands — what situation it sets up and what would make it fail if this change were wrong; a test that only repeats the new value catches later drift, not a value that is wrong today — say both halves. Nothing guards it → say so, say what check would prove it, and whether that can happen before it ships. When the question is intent rather than correctness (was removing this meant?), say the story or the author answers it.
  Only the human clears a must-review item; reviewer judgement, Codex consensus or green tests never do.
- Verify all requirements in the task are implemented — nothing missing, **nothing extra**: any change the task didn't ask for (adjacent "improvements", edits to working code, files outside the task's declared list) is an issue, not a bonus

**2. UI Verification (only if UI was changed)**
- Build the app using XcodeBuildMCP
- Launch in the simulator and navigate to the affected screen via normal app flow
- If normal flow cannot reach the changed UI yet (e.g., a prompt method not wired up), add a temporary trigger at the start of `Utils.runTest()` that drives the UI element, rebuild, and invoke runTest from the debug menu
- Take a screenshot of the affected screen
- Fetch the corresponding Figma design screenshot using Figma MCP
- Compare implementation against design — layout, colors, spacing, text, icons
- **Design waiver (no Figma node):** skip the Figma comparison and report it as `N/A — design waiver`; the screenshot, the regression check below, and the human's own look at the screenshot at the gate still apply
- **Regression check on existing elements:** the diff must not have removed or shifted anything on that screen the task didn't intend — verify pre-existing elements (badges, ticks, rows, decorations) are still present and correct, against the Figma design and/or the screen's prior state
- **Upgrade-path caveat:** a fresh simulator install cannot exercise upgrade-from-older-build state; if the task touches version-gated or upgrade-path logic, mark that path **UNVERIFIED** in the report — do not claim it passed
- **Before returning the report, remove any `Utils.runTest()` trigger you added and rebuild to confirm it still compiles.**
- Skip this check only if the task has no UI changes

**3. Codex Debate Review**

The reviewer debates with Codex directly to reach consensus on what issues exist. The reviewer only identifies issues; it does not fix code.

**Step 3a: Send to Codex**
- Send the diff and task intent to Codex with the Codex CLI: `codex review --uncommitted "<task intent>"`; follow-ups with `codex exec resume --last`
- Ask Codex to review for: bugs, logic errors, edge cases, missing error handling, style issues
- **Codex CLI unavailable → internal fallback, not a stop**: run the same review via Claude Code's built-in `/code-review` skill over the task's diff. Evaluate its findings with the same agree/disagree rigor as 3b (there is no reply thread — the caller's judgment replaces the debate loop), and record in the review report that the pass ran on the internal fallback. The adversarial pass itself is never skipped and never downgraded to UNVERIFIED.

**Step 3b: Evaluate each issue Codex raises**
- **Agree**: If the issue is valid, record it as a confirmed issue
- **Disagree**: If the issue is not valid (false positive, project convention, etc.), explain reasoning to Codex in the same thread

**Step 3c: Iterate until consensus**
- Continue in the same thread (`codex exec resume --last`) — don't start new sessions
- Debate until both sides agree on which issues are real
- Push back on over-engineered suggestions — the right fix is the minimal correct fix

**4. Verification**
- Execute exactly the task's **Verification** rows plus its **Re-run** list, read off the checklist entry — nothing else. Selection was made and confirmed at Step 5; do not add items by judging what the change "could affect", and never run the unfiltered test target
- Per method: **unit/integration rows** → scoped `test_sim`, one `-only-testing:Calorie Counter Tests/<TestClass>` entry in `extraArgs` per class; **script rows** → run the committed script from `verification/<feature>/` and assert against the row's criteria on the Proxyman capture when available, else on the script's own input/output (state which in the report) — but a row about a request the app itself sends cannot be observed without a capture: report it UNVERIFIED, never PASS; **review-only rows** → resolve inside check 3's adversarial pass, and the report cites that verdict as the row's evidence; **simulator-check rows** → check 2's UI verification is the evidence
- A red test, a script output that misses its criteria, or a review-only row check 3 could not clear is a confirmed issue. If the change plainly touches code no Verification/Re-run row covers, report that as a finding (the plan's list is incomplete) rather than silently widening the run
- **Match the run to the pass.** The first pass of a task runs the full scoped set above. A fix-loop pass re-runs only the rows the fix touched. A pass whose diff since the last review is only renames, comments or docs runs **no tests and no simulator**: compile the app and the test target (`build-for-testing`), `git grep` for stale names, and report checks 2 and 4 as "not run — rename/comment-only pass, compile verified". This is not UNVERIFIED — a Swift rename that missed a site fails to compile — but the pass must state which kind it was and why
- **No expectation drift.** Scan the diff for changes to existing tests, verification scripts, or their fixtures: an edited expected value, a weakened assertion (`assertEqual` → `assertNotNil`, a removed check), a deleted or skipped test, a re-recorded fixture. Each one must trace to a D-decision change in `design.html` or a changed row in the Verification Plan; otherwise it is a confirmed issue — the implementation was fitted to pass, not fixed. This applies with full force inside the review-fix loop, where the pressure to go green is highest
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

### Review Triage (opens the report, before any finding)
[Four lines: Intent · Expected scope · Actual scope (what falls outside expected scope, or "all in scope") · View (the git command that shows this change). Then Must review — one numbered item per layer, in the format the human gate presents (paragraph, pseudocode, why / ask / guard / where); Spot-check and Skip — one bullet per layer. "Nothing must-review" in one line when that is the case.]

### 1. Spec Compliance: PASS / ISSUES
### 2. UI Verification: PASS / ISSUES / UNVERIFIED / N/A
### 3. Codex Review: PASS / ISSUES
### 4. Verification: PASS / ISSUES / UNVERIFIED
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
2. **Show the human what they are approving — the review triage, in the conversation's language** (this is a message to the human, not an artifact). Open with four lines: **Intent**, **Expected scope**, **Actual scope**, **View** (the git command that shows the change). Then:
   - **Must review** — one numbered item per layer, each in this order:
     1. A short plain-words paragraph: what changed, and what it means for users or data.
     2. **Pseudocode in a code block**: what the code does when it runs, as steps and branches in order — who triggers it, what happens, what happens on success and on failure. Every step is a plain sentence in the conversation's language, never code: no assignments, comparisons, `guard`, `for` loops or function calls. It is not a field-by-field old → new list. Keep the real values that matter (a limit of 4, a key the user sees); name an identifier only where the reader needs it to find the code. Mark each changed step with a short `#` comment saying what it was before ("# new: was a fixed single reply"), and unchanged steps that matter with "# same as before". Cover only the path the change touches, about 15 short lines per block.
     3. A bullet list: **why** (the rule or sensitive area that made it must-review), **ask** (the one question the human answers), **guard** (plain words: what protects this change, or that nothing does and what would), **where** (`path:line`).
   - **Spot-check** and **Skip** — one bullet per layer: the layer, what changed or why it is skipped, its paths.
   - Verbatim, every item from the report's **Unverified** list. For UI tasks, the reviewer's simulator screenshot (path) beside the Figma reference so the human can eyeball the change themselves.

   A pseudocode block that reads like code — never this:

   ```
   submit(text, addressing):
     guard canSubmit
     for i in 0..<4:
       candidate == controlled → return
   ```

   The same path, the way it should read:

   ```
   User sends a message:
     not allowed while a reply is being generated or merged
     add the message to the conversation and start one relay round
   A relay round is at most 4 replies:
     ask the model for one reply
     look at who the model says speaks next:
       nobody, or the user → the round ends          # new: was always one reply
       another model role → note it is owed a turn, keep going
   When the round ends:
     an error mid-way → drop the unfinished reply, show Retry   # same as before
   ```

   An Approve must be informed consent; approving with must-review or UNVERIFIED items is an explicit, recorded acknowledgement.
   **What an Approve does NOT assert:** agent review plus green tests does not prove UI behaviour — QA's own testing and Step 7a's manual-QA list stay in the loop regardless of this Approve; say so when presenting a UI task.
3. **Call `AskUserQuestion`** with:
   - `question`: "Review for `<task name>` complete. Recommendation: APPROVE. Must review: `<items or none>`. Unverified: `<list or none>`. Approve commit + TaskUpdate?"
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
- UI verification compares against the task's Figma node — find it in the checklist entry or the story-analysis output; under a design waiver see check 2
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
| "The diff is too big to show, they trust me" | Show the triage: every must-review item with its pseudocode, plus the command to view the diff. Approval must be informed. |
| "Pasting the diff hunk is clearer than pseudocode" | The human reads the gate to decide, not to compile. Plain-sentence steps, changed ones marked. |
