# FatSecret Workflow Plugin

Claude Code plugin for the FatSecret delivery team — orchestrates the full feature development lifecycle from story analysis to branch completion.

## Installation

1. **Add the marketplace:**
   ```bash
   claude plugins marketplace add https://github.com/fatsecret-group/fatsecret-workflow.git
   ```

2. **Install the plugin:**
   ```bash
   claude plugins install fatsecret-workflow
   ```

3. **Verify it's working** — start Claude Code and run `/fatsecret-workflow:story-analysis` to check the skill loads.

**Update:**
```bash
claude plugin marketplace update fatsecret-workflow
claude plugin update fatsecret-workflow@fatsecret-workflow
```

**Uninstall:**
```bash
claude plugins uninstall fatsecret-workflow
claude plugins marketplace remove fatsecret-workflow
```

## Prerequisites

No other plugins are required — the workflow's verification rules, bug-diagnosis method, completion verification, and branch-finish procedure are built into the skills themselves.

### MCP servers and tools

Some skills integrate with external tools. The full feature-workflow checks them at Step 1 and reports a readiness table (MCP servers plus the Codex CLI): **XcodeBuildMCP is required** (and **Figma** when the feature has UI work) — a missing one is surfaced, never silently skipped. **Shortcut, Codex, Proxyman, and TestSecret are optional**, each with a defined fallback. Install what the skills you use need.

| MCP Server | Used by | Purpose | Missing → |
|------------|---------|---------|-----------|
| [XcodeBuildMCP](https://github.com/getsentry/XcodeBuildMCP) | `feature-workflow`, `review-task`, `validate-test-plan` | Build, scoped test runs, simulator UI automation | required — reported at Step 1 |
| [Figma MCP](https://developers.figma.com/docs/figma-mcp-server/) | `story-analysis`, `figma-driven-implementation`, `review-task` | Design-to-code workflow | required for UI features |
| [Shortcut MCP](https://www.npmjs.com/package/@shortcut/mcp) | `story-analysis`, `feature-workflow` | Read stories; search historical stories during design grilling; post the agreed decision summary to each story for QA | human pastes stories in / pastes the approved summary onto the stories |
| [Codex CLI](https://github.com/openai/codex) | `feature-workflow` (design challenge), `write-test-plan` (coverage challenge), `review-task` (review debate) | Independent adversary for the three review gates | built-in `/code-review` (review-task) or a fresh-context subagent (design & coverage challenges) |
| [TestSecret](https://claude.ai/code/artifact/656220b8-5cc5-46de-8880-e330839cbb62) | `write-test-plan`, `validate-test-plan` | Where QA keeps their test cases: Step 4 uploads the draft cases as Draft; Step 7a validates the feature's cases and writes results back | upload deferred with a note; Step 7a validates the dev side only |
| [Proxyman MCP](https://proxyman.com) | `review-task`, `validate-test-plan` | Network capture for script-based verification and runtime-only cases | scripts assert on their own input/output; capture-only cases stay punted to manual QA |

<details>
<summary><strong>MCP setup instructions</strong></summary>

#### Figma MCP

The skills call the official Figma MCP server (`get_design_context`, `get_screenshot`). Add it, then sign in with `/mcp`:

```bash
claude mcp add --transport http figma https://mcp.figma.com/mcp
```

#### Shortcut MCP

Shortcut now develops its MCP server only as a hosted server (`https://mcp.shortcut.com/mcp` — setup instructions there); the npm package below still works but is frozen. The skills name the npm package's tools (`stories-get-by-id`, `stories-search`, `stories-create-comment`, …).

```bash
claude mcp add shortcut -- npx -y @shortcut/mcp@latest
```

Set your API token:
```bash
export SHORTCUT_API_TOKEN=your_token_here
```

Or add to your project's `.mcp.json`:
```json
{
  "mcpServers": {
    "shortcut": {
      "command": "npx",
      "args": ["-y", "@shortcut/mcp@latest"],
      "env": {
        "SHORTCUT_API_TOKEN": "your_token_here"
      }
    }
  }
}
```

#### XcodeBuildMCP

Install via Homebrew:
```bash
brew install xcodebuildmcp
```

Then add to Claude Code:
```bash
claude mcp add xcodebuildmcp -- xcodebuildmcp mcp
```

#### Codex

Install the CLI and sign in:
```bash
npm install -g @openai/codex
codex login
```

The workflow calls the CLI directly (`codex exec --sandbox read-only`, `codex review`) — no MCP server to register.

</details>

## Skills

### Workflow orchestration

| Skill | Description |
|-------|-------------|
| `feature-workflow` | Main orchestrator — guides the full feature development lifecycle |

### Analysis & planning

| Skill | Description |
|-------|-------------|
| `story-analysis` | Analyze Shortcut stories + Figma designs into executable items |
| `writing-plans` | Turn the confirmed design into the Execution Checklist appended to `design.html` — one entry per task (files, verification rows, acceptance point) |
| `write-test-plan` | Write the Verification Plan (dev's own test considerations, one row per deliverable, appended to `design.html`) and a draft QA test plan (`test-plan.html`), uploaded to TestSecret as Draft for QA to own |

### Implementation

| Skill | Description |
|-------|-------------|
| `figma-driven-implementation` | Pixel-accurate UI implementation from Figma nodeIds |

### Review & quality

| Skill | Description |
|-------|-------------|
| `review-task` | Per-task review before every commit: review triage — the change sorted into must-review / spot-check / skip, each must-review item explained in plain-language pseudocode (sensitive areas from the project's `REVIEW.md`) — spec compliance, UI verification against Figma, Codex debate (built-in `/code-review` fallback), the task's declared verification, simplification — then a mandatory human approval gate |
| `validate-test-plan` | Validate what the per-task reviews left unproven (uncovered acceptance points, unverified rows) plus the feature's TestSecret cases — refute-first verdicts per sub-clause; criticality × verifiability routing shrinks the manual-QA list; runtime clauses captured via simulator + Proxyman or explicitly punted; outcome on QA's cases recorded to TestSecret as a run (Dev-Verified / AI-Tested / Failed / Retest / Skipped; punted cases stay Untested as QA's manual queue) |

## Per-project setup

- **`REVIEW.md`** (source root; may be git-excluded) — the project's "Sensitive areas" table: path patterns, why each is sensitive, and the question a reviewer must answer. `review-task` reads it and puts every touched area in front of the human before a commit. Without it, generic categories are used.
- **`docs/plans/CODEBASE-KNOWLEDGE.md`** — the codebase map the workflow reads before analysis, design, planning and build.
- **`verification/<feature>/`** — committed verification scripts and their fixtures.
- **Git worktrees** don't carry git-excluded files: copy `REVIEW.md`, the MCP config (`.mcp.json`) and build dependencies (e.g. `Pods/`) into a new worktree before starting.
