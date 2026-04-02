---
name: write-test-plan
description: Use when user provides stories or feature requirements and needs a test plan written, before writing implementation plans
---

## When to Use

- User provides stories, feature requirements, or a brief description of what to build
- BEFORE brainstorming or writing-plans
- The output test plan becomes the acceptance criteria for implementation

## Input

Stories can come in any form:
- A CSV file with short story descriptions
- A list of requirements in text
- A feature brief or spec document

Read and understand all stories before writing test cases.

**Input validation**: If the input is a story-analysis document, check that it includes a **Behavioral Dimensions** section. If missing, flag to the user that dimensions were not analyzed and ask whether to proceed or revisit story-analysis first.

## Output Format

Write the test plan as a CSV matching the format in `example-test-plan.csv` (in this skill's directory).

Read that file to understand the exact column structure, HTML formatting, and multi-row step pattern before writing.

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

1. Save the test plan to the feature's output directory (see feature-workflow Output Directory rule)
2. Present a summary to the user: total test cases, coverage areas, any gaps
3. Ask user to review and approve before proceeding to `writing-plans`
4. The approved test plan becomes the acceptance criteria for implementation
