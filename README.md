# FatSecret Workflow Plugin

Claude Code plugin for the FatSecret delivery team — orchestrates the full feature development lifecycle from story analysis to branch completion.

## Installation

1. **Add the marketplace:**
   ```bash
   claude plugins marketplace add https://github.com/fszbn/fatsecret-workflow.git
   ```

2. **Install the plugin:**
   ```bash
   claude plugins install fatsecret-workflow
   ```

3. **Verify it's working** — start Claude Code and run `/fatsecret-workflow:story-analysis` to check the skill loads.

**Uninstall:**
```bash
claude plugins uninstall fatsecret-workflow
claude plugins marketplace remove fszbn/fatsecret-workflow
```

## Prerequisites

This plugin depends on the **superpowers** plugin for core development skills (TDD, planning, debugging, etc.).

Install superpowers first:

```bash
claude plugins add superpowers-marketplace/superpowers
```

### Optional MCP servers

Some skills integrate with external tools via MCP. **Skills gracefully degrade if their MCP server is not configured** — you only need to install the ones you use.

| MCP Server | Used by | Purpose |
|------------|---------|---------|
| [Figma MCP](https://github.com/figma/figma-mcp) | `figma-design-brief`, `figma-driven-implementation`, `review-task` | Design-to-code workflow |
| [Codex MCP](https://github.com/openai/codex) | `review-task` | AI code review debates |
| [Shortcut MCP](https://www.npmjs.com/package/@shortcut/mcp) | `story-analysis` | Read stories from Shortcut |
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

### Optional plugins

These are useful companions but not bundled here — install them separately:

```bash
claude plugins add claude-plugins-official/code-simplifier
```

## Skills

### Workflow orchestration

| Skill | Description |
|-------|-------------|
| `feature-workflow` | Main orchestrator — guides the full feature development lifecycle |

### Analysis & planning

| Skill | Description |
|-------|-------------|
| `story-analysis` | Analyze Shortcut stories + Figma designs into executable items |
| `write-test-plan` | Generate test plans from stories/requirements |
| `figma-design-brief` | Fetch and analyze Figma designs into a design brief |

### Implementation

| Skill | Description |
|-------|-------------|
| `figma-driven-implementation` | Pixel-accurate UI implementation from Figma nodeIds |
| `run` | Build and run app — simulator (default) or `device`. Supports `--delete` for clean install. |

### Review & quality

| Skill | Description |
|-------|-------------|
| `review-task` | Per-task review: spec compliance + UI verification + Codex debate |

## Per-project setup

The `run` skill defaults to the first booted simulator and auto-discovers connected devices. Override workspace/scheme/bundle ID in your project's `.claude/skills/run/SKILL.md` if they differ from the defaults.

## Superpowers skills used

The `feature-workflow` skill orchestrates these superpowers skills:

- `superpowers:brainstorming`
- `superpowers:writing-plans`
- `superpowers:test-driven-development`
- `superpowers:verification-before-completion`
- `superpowers:finishing-a-development-branch`

If any of these are unavailable, the workflow will prompt you to install superpowers.
