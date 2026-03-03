# Multi-Agents Plugin for Claude Code

A Claude Code plugin that orchestrates multiple AI agents (Codex, Gemini, OpenCode) in parallel for code reviews and panel discussions.

## Skills

| Skill | Command | Description |
|-------|---------|-------------|
| **review-pr** | `/ma:review-pr <PR>` | Multi-agent PR review — launches 4 agents in parallel to review a GitHub PR, then synthesizes findings into a single comment |
| **round-table** | `/ma:round-table <topic>` | Multi-agent panel discussion — 4 AI panelists with distinct roles debate a topic across multiple rounds |

## Prerequisites

The following CLI tools must be installed and authenticated:

| Tool | Install | Purpose |
|------|---------|---------|
| [Codex](https://github.com/openai/codex) | `npm install -g @openai/codex` | OpenAI's coding agent |
| [Gemini CLI](https://github.com/google-gemini/gemini-cli) | `npm install -g @anthropic-ai/gemini-cli` | Google's Gemini agent |
| [OpenCode](https://github.com/opencode-ai/opencode) | See repo for install | Multi-model coding agent (GLM-5, Kimi-K2.5) |
| [GitHub CLI](https://cli.github.com/) | `brew install gh` | Required by review-pr for PR operations |

Skills degrade gracefully — if a tool is missing, the skill skips it and continues with the others.

## Installation

### From GitHub (recommended)

```bash
claude plugin marketplace add quabug/multi-agents-plugin
claude plugin install ma@multi-agents-plugin
```

### From local path

```bash
claude plugin marketplace add /path/to/this/repo
claude plugin install ma@local
```

## Usage

### PR Review

Review a GitHub PR with multiple AI agents in parallel:

```bash
/ma:review-pr 42
/ma:review-pr https://github.com/owner/repo/pull/42
```

**What it does:**
1. Fetches the PR diff and existing review comments
2. Launches Codex, Gemini, GLM-5, and Kimi-K2.5 in parallel — each reviews independently
3. Waits for all agents, then synthesizes findings by severity (critical/major/minor)
4. Posts a single consolidated review comment on the PR via `gh`

Existing review comments are fed to each agent so they don't duplicate prior feedback.

### Round Table

Host a structured multi-round panel discussion:

```bash
/ma:round-table "Should we use ECS or traditional OOP for our game engine?"
/ma:round-table "Microservices vs monolith for our new project"
```

**What it does:**
1. Assigns distinct roles to 4 panelists (Codex, Gemini, OpenCode, Claude) from a [role catalog](plugins/ma/skills/round-table/references/roles.md)
2. Runs multi-round debates — each round poses a focused question
3. Claude participates as a panelist AND moderates, synthesizing each round
4. Saves a full markdown transcript to the working directory
5. User controls pacing: continue, steer, or stop after each round

## Plugin Structure

```
.claude-plugin/
└── marketplace.json          ← Marketplace manifest
plugins/
└── ma/
    ├── .claude-plugin/
    │   └── plugin.json       ← Plugin manifest
    └── skills/
        ├── review-pr/
        │   └── SKILL.md      ← PR review skill
        └── round-table/
            ├── SKILL.md      ← Round-table skill
            └── references/
                └── roles.md  ← 20+ role catalog for panelists
```

## License

MIT
