# Multi-Agents Plugin for Claude Code

A Claude Code plugin that orchestrates configurable AI agents in parallel for code reviews and panel discussions. Supports Codex, Gemini, OpenCode, and any future CLIs via the agent catalog.

## Skills

| Skill | Command | Description |
|-------|---------|-------------|
| **ask** | `/ask <question>` | Ask all configured agents a question in parallel and display their responses with a brief synthesis |
| **review-pr** | `/review-pr <PR>` | Multi-agent PR review — launches configured agents in parallel to review a GitHub PR, then synthesizes findings into a single comment |
| **round-table** | `/round-table <topic>` | Multi-agent panel discussion — configured AI panelists with distinct roles debate a topic across multiple rounds |

## Configuration

Configure which agents to use via a `## Multi-Agents` section in your CLAUDE.md (project-level or user-level):

```markdown
## Multi-Agents
- codex
- gemini
- opencode: bailian-coding-plan/glm-5
- opencode: bailian-coding-plan/kimi-k2.5
```

**Syntax:** `- {cli-name}` or `- {cli-name}: {model}`

- Same CLI can appear multiple times with different models
- Lines not matching the pattern are ignored
- Project CLAUDE.md takes precedence over user CLAUDE.md

### Auto-Detection Fallback

If no `## Multi-Agents` section is found, the plugin auto-detects available CLIs:

```bash
which codex gemini opencode
```

One default entry (no model override) is added for each CLI found on `$PATH`. Skills degrade gracefully — if a tool is missing, the skill skips it and continues with the others.

### Supported Agents

| Agent | Binary | Install | Notes |
|-------|--------|---------|-------|
| [Codex](https://github.com/openai/codex) | `codex` | `npm install -g @openai/codex` | OpenAI's coding agent |
| [Gemini CLI](https://github.com/google-gemini/gemini-cli) | `gemini` | `npm install -g @anthropic-ai/gemini-cli` | Google's Gemini agent |
| [OpenCode](https://github.com/opencode-ai/opencode) | `opencode` | See repo for install | Multi-model agent (supports `-m model` flag) |

See [`references/agent-catalog.md`](plugins/multi-agents/references/agent-catalog.md) for full CLI details, command templates, and instructions for adding new agents.

## Prerequisites

| Tool | Install | Purpose |
|------|---------|---------|
| [GitHub CLI](https://cli.github.com/) | `brew install gh` | Required by review-pr for PR operations |

Plus at least one configured or auto-detected agent from the supported list above.

## Installation

```bash
claude plugin marketplace add quabug/multi-agents-plugin
claude plugin install multi-agents@multi-agents-plugin
```

## Usage

### PR Review

Review a GitHub PR with multiple AI agents in parallel:

```bash
/review-pr 42
/review-pr https://github.com/owner/repo/pull/42
```

**What it does:**
1. Resolves the agent roster (from config or auto-detection)
2. Fetches the PR diff and existing review comments
3. Launches all configured agents in parallel — each reviews independently
4. Waits for all agents, then synthesizes findings by severity (critical/major/minor)
5. Posts a single consolidated review comment on the PR via `gh`

Existing review comments are fed to each agent so they don't duplicate prior feedback.

### Ask

Ask all configured agents a question in parallel:

```bash
/ask "What are the pros and cons of using ref structs in C#?"
/ask "How should I structure error handling in this codebase?"
```

**What it does:**
1. Resolves the agent roster (from config or auto-detection)
2. Sends the question to all agents in parallel
3. Displays each agent's response with clear labels
4. Provides a brief synthesis highlighting agreements and divergences

### Round Table

Host a structured multi-round panel discussion:

```bash
/round-table "Should we use ECS or traditional OOP for our game engine?"
/round-table "Microservices vs monolith for our new project"
```

**What it does:**
1. Resolves the agent roster and generates distinct roles tailored to the topic
2. Runs multi-round debates — each round poses a focused question
3. Claude participates as a panelist AND moderates, synthesizing each round
4. Saves a full markdown transcript to the working directory
5. User controls pacing: continue, steer, or stop after each round

## Plugin Structure

```
.claude-plugin/
└── marketplace.json          <- Marketplace manifest
plugins/
└── multi-agents/
    ├── .claude-plugin/
    │   └── plugin.json       <- Plugin manifest
    ├── references/
    │   └── agent-catalog.md  <- Agent CLI catalog (shared by all skills)
    └── skills/
        ├── ask/
        │   └── SKILL.md      <- Ask all agents skill
        ├── review-pr/
        │   └── SKILL.md      <- PR review skill
        └── round-table/
            └── SKILL.md      <- Round-table skill (roles generated at runtime)
```

## License

MIT
