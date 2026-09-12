# Agent Resolution

Build the roster from the task's named agents/models, or a `## Multi-Agents` section in the
applicable project/user `CLAUDE.md`. Project configuration takes precedence; entries are
`- cli` or `- cli: model`. Preserve model identifiers exactly.

When a multi-agent consultation is requested and no roster is configured, discover available
`codex`, `gemini`, `opencode`, `pi`, and `qwen` executables. Use their configured defaults.
Apply target filters and `--skip <name>` before loading CLI details; match CLI, model suffix,
`cli:model`, or display name case-insensitively. Keep distinct entries for separate models.

Load [agent-catalog.md](agent-catalog.md), then only the reference for each selected CLI:
[codex](codex.md), [gemini](gemini.md), [opencode](opencode.md), [pi](pi.md), [qwen](qwen.md).
Verify executable availability and report missing explicitly requested participants. Display
CLI plus model when specified so answers are attributable. Do not substitute guessed models.
