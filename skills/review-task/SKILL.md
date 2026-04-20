---
name: review-task
description: Use after implementing a task — runs spec compliance, build, simulator run, UI verification against Figma, and codex debate review, then returns APPROVED or ESCALATED. Does NOT commit or mark the task complete; the caller does that in the plan's Step 3.
---

## When to Use

- After an implementation task compiles cleanly (plan Step 1 finished), before Step 3 commits
- Before the caller commits and calls `TaskUpdate(completed)`

## Contract

This skill:
- Reviews spec compliance, Figma UI match, and Codex debate
- Builds the app and runs the simulator for UI verification
- If the changed UI cannot be reached via normal app flow, autonomously adds a temporary trigger at the start of `Utils.runTest()`, exercises it from the debug menu, screenshots, then removes the trigger before returning
- Runs an internal review-fix loop (caps at 3 iterations)
- Gates on human confirmation
- Returns a verdict: `APPROVED` or `ESCALATED`
- **Human gate is mandatory in every mode, including auto.** Auto mode reduces LLM-initiated questions, but the final human gate in this skill is a harness block, not a courtesy. Do not rationalize past it.

Commit + task completion are the caller's responsibility, executed as the plan's Step 3 only when this skill returns `APPROVED`.

## Process

Dispatch a single reviewer subagent with the following instructions. The reviewer handles all three checks in one pass. It does NOT fix code — it only identifies and reports issues.

### Reviewer Subagent Instructions

The reviewer performs these checks in order:

**1. Spec Compliance**
- Read the task description from the implementation plan
- Read the changed files (use git diff)
- Verify all requirements in the task are implemented — nothing missing, nothing extra

**2. UI Verification (only if UI was changed)**
- Build the app using XcodeBuildMCP
- Launch in the simulator and navigate to the affected screen via normal app flow
- If normal flow cannot reach the changed UI yet (e.g., a prompt method not wired up), add a temporary trigger at the start of `Utils.runTest()` that drives the UI element, rebuild, and invoke runTest from the debug menu
- Take a screenshot of the affected screen
- Fetch the corresponding Figma design screenshot using Figma MCP
- Compare implementation against design — layout, colors, spacing, text, icons
- **Before returning the report, remove any `Utils.runTest()` trigger you added and rebuild to confirm it still compiles.**
- Skip this step if the task has no UI changes

**3. Codex Debate Review**

The reviewer debates with Codex directly to reach consensus on what issues exist. The reviewer only identifies issues; it does not fix code.

**Step 3a: Send to Codex**
- Use `mcp__codex__codex` to send the diff and task intent
- Ask Codex to review for: bugs, logic errors, edge cases, missing error handling, style issues

**Step 3b: Evaluate each issue Codex raises**
- **Agree**: If the issue is valid, record it as a confirmed issue
- **Disagree**: If the issue is not valid (false positive, project convention, etc.), explain reasoning to Codex using `mcp__codex__codex-reply`

**Step 3c: Iterate until consensus**
- Use `mcp__codex__codex-reply` to continue the conversation
- Debate back and forth until both sides agree on which issues are real
- Keep the thread — use codex-reply, don't start new sessions
- Push back on over-engineered suggestions — the right fix is the minimal correct fix
- This loop continues until mutual agreement on the issue list

**Important: The reviewer NEVER fixes code. It only debates with Codex to produce an agreed-upon list of real issues.**

### Output Format

The reviewer compiles all findings into a single report:

```
## Post-Task Review: [Task Name]

### 1. Spec Compliance: PASS / ISSUES
[Details]

### 2. UI Verification: PASS / ISSUES / SKIPPED
[Details]

### 3. Codex Review: PASS / ISSUES
[List of agreed issues with severity]

### Recommendation to human: APPROVE / ESCALATE
[Summary. This is the reviewer subagent's INPUT to the human gate — it is NOT the skill's final verdict. The human sets the verdict in the handoff step below.]
```

### Review-Fix Loop

If issues are found:
1. Reviewer reports all issues to the main agent
2. Main agent (or implementer subagent) fixes the issues — **fixes only; no commit**
3. Main agent re-invokes this skill to review again
4. Repeat until APPROVED or 3 iterations reached

If 3 iterations pass without APPROVED, return `ESCALATED` with the outstanding issues and stop. The caller must surface to the human and wait for instructions — it must NOT proceed to commit.

### Verdict and Handoff (mandatory human gate)

After the loop resolves, the reviewer subagent has produced a Recommendation. **The skill's verdict is set by the human, not by the reviewer.**

1. Post a concise summary to the user (what was implemented, what was reviewed, fixes made, reviewer's recommendation).
2. **Call `AskUserQuestion`** with:
   - `question`: "Review for `<task name>` complete. Recommendation: `<APPROVE|ESCALATE>`. Approve commit + TaskUpdate?"
   - `header`: "Task review"
   - `options`:
     - `"Approve"` — proceeds to commit and TaskUpdate
     - `"Escalate"` — surface issues, block commit, wait for instructions
   - `multiSelect`: false
3. Map the user's answer to the verdict:
   - User selected `Approve` → return `APPROVED` to the caller. Caller proceeds to plan Step 3 (commit + `TaskUpdate(completed)`).
   - User selected `Escalate`, picked "Other" with a non-approve response, or typed any message that isn't unambiguous approval → return `ESCALATED`. Caller stops and waits for instructions.

**You MUST call `AskUserQuestion` before returning a verdict.** The call is the gate. A reviewer subagent saying "APPROVE" is not a verdict — it is a recommendation that feeds into the human gate. Do NOT infer, assume, or synthesize the human's answer from prior messages, auto mode, or the reviewer's recommendation.

If 3 iterations pass without reaching an APPROVE recommendation, call `AskUserQuestion` with the outstanding issue list and options `"Retry with guidance"` / `"Escalate (block)"`. Map `Retry` to a fresh review cycle; map `Escalate` to `ESCALATED`.

Commit + task completion belong exclusively to the plan's Step 3, executed by the caller only when this skill returns `APPROVED`.

## Important

- The reviewer subagent only reports issues; the implementer subagent only writes code; this skill only produces a verdict
- UI verification requires a Figma URL — check the implementation plan or story-analysis output for the relevant Figma node
- If the app cannot be built (e.g., worktree without Xcode project), skip UI verification and note it in the report
- For the Codex debate: respond to every issue Codex raises, be willing to debate or concede, keep it simple

## Red Flags — stop and call `AskUserQuestion`

If any of these thoughts appears before you return a verdict, you are skipping the human gate. Fix it by calling `AskUserQuestion` now.

| Thought | Reality |
|---|---|
| "Reviewer said APPROVE, I can commit" | Reviewer's output is a RECOMMENDATION. The gate is `AskUserQuestion`. |
| "Auto mode means I don't need to ask" | Auto mode does not disable Rigid-skill gates. |
| "User said 'go' earlier, that covers this" | `go` authorizes executing the plan, not skipping a per-task human gate. |
| "This task was trivial, confirmation would be noise" | Then the user will click Approve in one tap. Call it anyway. |
| "I'll batch gates at the end of the plan" | Every task's gate is independent. Don't batch. |
