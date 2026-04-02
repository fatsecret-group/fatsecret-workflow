---
name: review-task
description: Use after completing an implementation task — runs spec compliance, UI verification against Figma, and codex debate review in a single pass
---

## When to Use

- After an implementation task is marked as completed (triggered by TaskCompleted hook)
- Before presenting results to the human for confirmation

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
- Take a screenshot of the affected screen
- Fetch the corresponding Figma design screenshot using Figma MCP
- Compare implementation against design — layout, colors, spacing, text, icons
- Skip this step if the task has no UI changes

**3. Codex Debate Review**

This replaces the old `codex-review` skill call. The reviewer debates with Codex directly to reach consensus on what issues exist — but does NOT fix any code.

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

### Verdict: APPROVED / ISSUES
[Summary]
```

### Review-Fix Loop

If issues are found:
1. Reviewer reports all issues to the main agent
2. Main agent (or implementer subagent) fixes the issues
3. Main agent re-invokes this skill to review again
4. Repeat until APPROVED or 3 iterations (then escalate to human)

### After Review Passes

1. Present results to the human — summarize what was implemented, what was reviewed, any fixes made
2. Wait for human confirmation
3. Only after human says ok → commit the changes
4. **Mark the task as completed** (TaskUpdate status=completed)

**Do NOT commit during implementation or review.** Code is only committed after human verification.

**This skill is the ONLY mechanism that marks tasks complete.** The implementer and the main agent must never call TaskUpdate(completed) directly — only review-task does, after all checks pass and human confirms.

## Important

- The reviewer subagent must NOT fix code — only report issues
- The implementer subagent must NOT commit — only write code
- UI verification requires Figma URL — check the implementation plan or story-analysis output for the relevant Figma node
- If the app cannot be built (e.g., worktree without Xcode project), skip UI verification and note it in the report
- For the Codex debate: always respond to every issue Codex raises, be willing to debate or concede, keep it simple
