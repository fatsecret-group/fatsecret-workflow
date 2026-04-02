---
name: story-analysis
description: Use when starting a feature with Stories (from Shortcut, text, etc.) and optionally Figma designs — analyzes stories, explores codebase for hidden associations, and produces structured executable items for downstream test plans and coding plans
---

## When to Use

- At the start of feature development, BEFORE `write-test-plan` or `writing-plans`
- When the user provides Stories (Shortcut IDs, Iteration IDs, or pasted text)
- When Figma design URLs are available alongside Stories

## Core Objective

Transform raw Stories and Figma designs into logically complete, executable items — each with enough context (preconditions, related code, affected existing logic) so that:

- A developer (or AI) can fully implement an item without missing hidden dependencies
- A tester (or AI) can design a complete test flow without overlooking affected existing functionality

When a Story is a small isolated change, the output reflects that. But when hidden relationships exist, the Skill must proactively discover and surface them.

## Mindset: Picky Analyst

Approach every Story with a picky, adversarial mindset. Your job is NOT to accept requirements at face value — it is to find every edge case, ambiguity, and gap that could cause confusion or inconsistent understanding between product, design, dev, and QA.

Specifically, watch for:
- **Ambiguous trigger conditions**: "after the first X" — first ever? First in this session? First since upgrade? First since a flag was reset?
- **Undefined user segments**: Stories often describe the happy path for one user type. Actively ask: what about other segments — existing users upgrading, guest users, users who already completed this action before the feature existed?
- **Inconsistent terminology**: Does the same word mean the same thing across all Stories? When two Stories use different words, do they mean the same thing or different things?
- **Missing state transitions**: If state A leads to state B, what happens if the user is already in state B when the feature ships?
- **Cross-Story contradictions**: When Story A says one thing and Story B implies another, verify the intended behavior is explicitly defined
- **Upgrade path gaps**: If a feature introduces new flags or counters, what is the initial value for existing users who upgrade? Does the feature silently skip them, or does it behave incorrectly?

Every ambiguity you find must appear in the output — either as a resolved clarification or as an explicit open question.

## Input

| Input | Source | Required? |
|-------|--------|-----------|
| Stories | Story ID, Iteration ID, or user-provided text | Yes |
| Figma design(s) | Figma URL | No (but must be analyzed if available) |
| Additional context | User-provided verbal context | No |

Supports both single Story and batch Stories (a Feature).

## Process

### Phase 1: Explore & Analyze

**Step 1 — Fetch current Stories**
Fetch the Stories the user provided — these are the Stories you are analyzing:
- If given an **Iteration ID or URL**: call `mcp__shortcut__iterations-get-stories` to get the story list, then `mcp__shortcut__stories-get-by-id` for each
- If given **Story IDs**: call `mcp__shortcut__stories-get-by-id` for each
- If given **pasted text**: read directly
- If Figma URLs are provided by the user, fetch design screenshots and context
- If multiple Stories, identify relationships and ordering between them

**Step 1b — Extract & fetch Figma designs from Stories**
After fetching all Stories, scan every Story description for Figma URLs (patterns like `figma.com/board/...`, `figma.com/design/...`, `figma.com/file/...`). Collect all unique URLs found.

1. If Figma URLs are found in Story descriptions: present them to the user grouped by Story for confirmation, then fetch design screenshots and context via Figma MCP for each confirmed URL
2. If NO Figma URLs are found anywhere (neither in Stories nor provided by the user) BUT the Stories describe UI elements (screens, banners, popups, etc.): ask the user whether Figma designs exist for this feature. Do not proceed past Step 3 without design coverage for UI-heavy features
3. Combine user-provided Figma URLs (if any) with those extracted from Stories — deduplicate by fileKey+nodeId

This step is critical because product teams often embed design links directly in Story descriptions rather than providing them separately.

**Step 2 — Search for relevant PAST Stories (separate from Step 1)**
This is a **different operation** from Step 1. Step 1 fetched the *current* Stories you are analyzing. Step 2 searches Shortcut for *other, previously completed* Stories that provide historical context — prior implementations, related features, past decisions that affect this work.

Why this matters: past Stories reveal what was already tried, what constraints exist, and what related systems were touched. Without this context, you risk duplicating existing work, contradicting past decisions, or missing dependencies that only show up in historical context.

1. Based on the current Stories' intent, **suggest 2-4 search keywords** (e.g. feature names, UI element names, domain terms) and present them as a checklist using `AskUserQuestion` with selectable options (all ticked by default). The user can untick keywords they don't want or add new ones
2. After the user confirms or adjusts keywords, search using `mcp__shortcut__stories-search` with each keyword:
   - Use the `name` parameter for keyword matching
   - Use `label: "iOS"` to filter to iOS stories only
   - Use `isDone: true` to focus on completed past work
   - Exclude bug stories — only include `feature` and `chore` story types. Bug context is useful but should come from code/git history, not from story search results
3. Present the search results as a concise list (Story ID, title, state) grouped by keyword
4. Let the user select which past Stories are relevant — fetch full details for selected ones and incorporate their context into subsequent analysis steps

> **Do not confuse Step 1 with Step 2.** Fetching the current iteration's stories (Step 1) is not the same as searching for historical context (Step 2). Even if you already called Shortcut MCP in Step 1, you must still perform the keyword search in Step 2. They use different API calls for different purposes.

**Step 3 — Extract intent**
- From each Story: what is being added, changed, or removed
- From Figma designs (if available): UI elements, interaction flows, new or changed screens
- Cross-reference Story text with Figma design; flag any inconsistencies

**Step 4 — Explore code associations & disambiguate terms**
If a codebase is available (i.e. the skill is running within a code project):
- Based on the intent extracted from Stories, search the codebase for related code
- For each discovered association, record: location, current function, why it may be affected
- **Term disambiguation**: For every domain-specific term in the Stories (UI element names, feature names, action verbs), search the codebase for all implementations that match or relate to that term. If multiple distinct implementations are found for the same term, collect them for the Confirmation Gate — ask the human which one the Story refers to and whether the others are also affected.

If no codebase is available, inform the user and ask for confirmation to proceed without code exploration. If confirmed, skip this step and continue — the output will note that code associations were not analyzed.

**Step 5 — Discover behavioral dimensions & analyze segments**
Based on the Stories and code explored so far, identify **behavioral dimensions** — factors specific to this feature that could cause it to behave differently for different users or contexts. Derive them from what the Stories and code reveal, not from a fixed checklist. For example, account type (free vs premium) is a common dimension that affects feature availability.

Present the discovered dimensions to the human and ask: "Based on these Stories, I identified these dimensions that affect this feature's behavior: [list]. Are there others I'm missing?"

Then, for each relevant user segment (e.g. new users, existing users upgrading, returning users — adapt to the feature), determine:
- What is the initial state of any newly introduced flags, counters, or persisted values?
- Which parts of the feature are reachable vs unreachable?
- Are there unintended behaviors caused by pre-existing state conflicting with new logic?

**Step 6 — Present preliminary analysis**
Present to the user:
- Feature flows and affected areas discovered from the codebase, organized by functionality (not raw code locations)
- Related existing behavior that may need to change or be tested alongside the new work
- **User segment impact**: How each relevant segment is affected
- Inconsistencies between Stories and Figma designs (if any)
- **Missing designs**: If any UI element referenced in Stories cannot be found in the provided Figma designs, explicitly flag it as missing and ask the user for the design. Do NOT proceed with incomplete design coverage.

Then proceed to the Confirmation Gate to resolve ambiguities interactively.

### Human Confirmation Gate

Walk the user through questions **one at a time**, in this order:
1. **Term disambiguations** (from Step 4): "Story says X — I found these in the code: [list]. Which one is meant? Are the others also affected?"
2. **Dimension completeness** (from Step 5): "I identified these dimensions that affect this feature: [list]. Are there others I'm missing?"
3. **Ambiguities and edge cases**: For each, state what is unclear and why it matters. Provide options if applicable.

For each question:
1. State it clearly
2. Provide options if applicable
3. Wait for the user's answer before moving to the next one

After all questions are resolved, present a brief summary of confirmed decisions and ask the user to confirm before proceeding to Phase 2.

Do NOT dump all questions as a list for the user to answer in bulk — this leads to skipped questions and incomplete answers.

### Phase 2: Organize & Output

**Step 7 — Organize executable items**
- Based on confirmed information, split or merge requirements into logically complete Items
- Each Item contains: summary, preconditions, scope of impact, key UI specs, upgrade impact, test considerations
- Items are not necessarily 1:1 with Stories — one Story may become multiple Items, multiple Stories may merge into one Item
- Arrange Items in execution order respecting dependencies

**Step 8 — Save document**
- Save to the feature's output directory (see feature-workflow Output Directory rule)
- Present a summary to the user
- This document is the input for `write-test-plan` and `writing-plans`

## Output Format

```markdown
# Feature: <feature name>

## Overview (batch mode only)
- Feature description
- Dependencies and execution order between Stories

## Behavioral Dimensions
Table listing each dimension that affects this feature's behavior, its possible values, and which Items it impacts. Confirmed by human during analysis.

## User Segments & Upgrade Impact
Table or list describing each relevant user segment: who they are, what pre-existing state they may have, and how the feature behaves for them on first encounter.

## Executable Items

### Item 1: <title>
**Source**: Story #xxx / Figma node xxx
**Summary**: What to do and why
**Preconditions**: What existing logic or other Items this depends on
**Scope of Impact**:
  - New functionality to add
  - Existing functionality to modify (confirmed by human)
  - Existing functionality to remove (confirmed by human)
**Key UI Specs**: (if Figma available, list design parameters)
**Upgrade Impact**: How this item behaves for existing users updating the app — initial flag states, reachability, edge cases
**Test Considerations**: Complete test thinking for this Item, explicitly including upgrade scenarios

## Ambiguities & Open Questions
Numbered list of every unresolved ambiguity, edge case, or definition gap discovered during analysis. Each entry should describe: the ambiguity, why it matters, and which Items it affects.
```

## Important

- Do NOT skip the Human Confirmation Gate — presenting associations and waiting for confirmation is mandatory
- The output document must be self-contained: a reader should understand the full scope without referring back to the original Stories
- Every Item MUST include an **Upgrade Impact** section — even if the answer is "no impact on existing users"
- The **Ambiguities & Open Questions** section is mandatory — if you found zero ambiguities, you didn't look hard enough
