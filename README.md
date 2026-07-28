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
/plugin update fatsecret-workflow
```

**Uninstall:**
```bash
claude plugins uninstall fatsecret-workflow
claude plugins marketplace remove fatsecret-group/fatsecret-workflow
```

## Prerequisites

No other plugins are required — the workflow's TDD rules, bug-diagnosis method, completion verification, and branch-finish procedure are built into the skills themselves.

### Optional MCP servers

Some skills integrate with external tools via MCP. **In the full feature-workflow, the Codex, Figma, and XcodeBuildMCP gates are required** — the workflow stops and asks rather than silently skipping them, unless a skill explicitly defines a fallback. Install the servers for the skills you use.

| MCP Server | Used by | Purpose |
|------------|---------|---------|
| [Figma MCP](https://github.com/figma/figma-mcp) | `story-analysis`, `figma-driven-implementation`, `review-task` | Design-to-code workflow |
| [Codex MCP](https://github.com/openai/codex) | `review-task` | AI code review debates |
| [Shortcut MCP](https://www.npmjs.com/package/@shortcut/mcp) | `story-analysis`, `feature-workflow` | Read stories from Shortcut; search historical stories during design grilling |
| [XcodeBuildMCP](https://github.com/getsentry/XcodeBuildMCP) | `run`, `review-task` | Build, run, UI automation |

<details>
<summary><strong>MCP setup instructions</strong></summary>

#### Figma MCP

Figma MCP is an Anthropic built-in integration. Enable it in Claude Code:

```bash
claude mcp add figma -- npx -y figma-developer-mcp --figma-api-key=YOUR_FIGMA_TOKEN
```

Or get a token at https://www.figma.com/developers/api#access-tokens

#### Shortcut MCP

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
claude mcp add xcodebuildmcp -- xcodebuildmcp
```

#### Codex MCP

Install globally:
```bash
npm install -g @openai/codex
```

Then add to Claude Code:
```bash
claude mcp add codex -- codex --mcp
```

Requires `OPENAI_API_KEY` environment variable.

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
| `write-test-plan` | Generate test plans from stories/requirements; on approval, uploads the manual QA cases to TestSecret (`testsecret` MCP) as Draft — the authoritative case library |

### Implementation

| Skill | Description |
|-------|-------------|
| `figma-driven-implementation` | Pixel-accurate UI implementation from Figma nodeIds |
| `run` | Build and run app — simulator (default) or `device`. Supports `--delete` for clean install. |

### Review & quality

| Skill | Description |
|-------|-------------|
| `review-task` | Per-task review: spec compliance + UI verification + Codex debate (built-in `/code-review` fallback when Codex is unavailable) |
| `validate-test-plan` | Validate a test plan against the implementation — refute-first verdicts per sub-clause; criticality × verifiability routing shrinks the manual-QA list; runtime clauses captured via simulator + Proxyman or explicitly punted; outcome recorded to TestSecret as a run (Dev-Verified / AI-Tested / Failed / Retest / Skipped; punted cases stay Untested as QA's manual queue) |

## Per-project setup

The `run` skill defaults to the first booted simulator and auto-discovers connected devices. Override workspace/scheme/bundle ID in your project's `.claude/skills/run/SKILL.md` if they differ from the defaults.
