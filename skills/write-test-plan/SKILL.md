---
name: write-test-plan
description: Use when user provides stories or feature requirements and needs a test plan written, before writing implementation plans
---

## When to Use

- User provides stories, feature requirements, or a brief description of what to build
- After `story-analysis`, BEFORE `writing-plans`
- The output test plan becomes the acceptance criteria for implementation

## Input

Stories can come in any form:
- A CSV file with short story descriptions
- A list of requirements in text
- A feature brief or spec document

Read and understand all stories before writing test cases.

**Input validation**: If the input is a story-analysis document, check that it includes a **Behavioral Dimensions** section. If missing, flag to the user that dimensions were not analyzed and ask whether to proceed or revisit story-analysis first.

## Output Format

Write the test plan as **self-contained HTML with a left-side sticky TOC** — copy the structure from the shared skeleton at `../feature-workflow/references/plan-doc-skeleton.html` (in the `fatsecret-workflow` plugin). Render the cases as one or more `<table>`s grouped by feature area (see Grouping below); use the `--pass-*` / `--warn-*` / `--gate-*` CSS vars from the skeleton for status cells if you include a result column.

Use `example-test-plan.csv` (in this skill's directory) as the reference for the **column structure and the multi-row step pattern** — map those columns to table headers and the step rows to table rows.

**CSV alternative:** If the project's CLAUDE.md asks for CSV, or QA needs to import the plan into a spreadsheet, emit `test-plan.csv` matching `example-test-plan.csv` instead. HTML is the default; CSV is the explicit-request fallback.

## Writing Guidelines

### Test Case Coverage

For each story/feature, generate test cases covering:
- **Happy path**: Normal user flow works correctly
- **UI verification**: Visual elements are present and correct
- **Entry points**: All ways to reach the feature (tabs, buttons, notifications, widgets, deep links, 3D Touch)
- **Account types**: Both Member and Premium Member behavior
- **Market restrictions**: Features unavailable in certain markets
- **Edge cases**: Empty states, no results, first-time users, returning users
- **Navigation**: Back button, close button, dismissing flows
- **Persistence**: State preserved after navigation, reinstall, app update
- **Data integrity**: Items saved/logged correctly

### Naming Convention

Titles should follow: `[N] Verify [specific observable outcome] when [trigger/condition]`

Examples:
- `[1] Verify Food logging bottom sheet is opening when Add Food is tapped from Diary meal type`
- `[9] Verify UI of the Launcher bottom sheet`
- `[39] Verify Food is recorded correctly in Diary`

### Grouping

Order test cases logically by feature area:
1. Entry points and navigation
2. UI verification
3. Core functionality
4. Edge cases and error states
5. Account/market-specific behavior
6. Persistence and data integrity

## After Writing

1. Save the test plan to the feature's output directory (see feature-workflow Output Directory rule) as `test-plan.html` (or `test-plan.csv` per the fallback above)
2. Present a summary to the user: total test cases, coverage areas, any gaps
3. Ask user to review and approve before proceeding to `writing-plans`
4. The approved test plan becomes the acceptance criteria for implementation
