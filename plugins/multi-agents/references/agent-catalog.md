# Agent Catalog

Single source of truth for all supported external CLI agents. All skills read this file during setup to build commands for the resolved agent roster.

---

## Configuration Format

Users configure which agents to use via a `## Multi-Agents` section in their CLAUDE.md (project or global):

```markdown
## Multi-Agents
- codex
- gemini
- opencode: bailian-coding-plan/glm-5
- opencode: bailian-coding-plan/kimi-k2.5
```

**Syntax:** `- {cli-name}` or `- {cli-name}: {model}`

- Same CLI can appear multiple times with different models.
- Lines not matching `- {word}` pattern are ignored.
- The section ends at the next `##` heading or end of file.

### Display Name Rules

| Entry | Display Name |
|---|---|
| `codex` | Codex |
| `gemini` | Gemini |
| `opencode` | OpenCode |
| `opencode: bailian-coding-plan/glm-5` | OpenCode (GLM-5) |
| `opencode: bailian-coding-plan/kimi-k2.5` | OpenCode (Kimi-K2.5) |

**Rule:** Capitalize the CLI name. If a model is specified, append the last segment of the model path in parentheses, uppercased per segment (split on `-`, capitalize first letter of each word, rejoin with `-`).

### Fallback: Auto-Detection

If no `## Multi-Agents` section is found in any CLAUDE.md, auto-detect available CLIs:

```bash
which codex gemini opencode 2>&1
```

For each CLI found on `$PATH`, add one default entry (no model override). This means auto-detection produces at most one entry per CLI binary.

---

## Supported Agents

### Codex (OpenAI)

- **Binary:** `codex`
- **Install:** `npm install -g @openai/codex`
- **Approval mode:** `--full-auto` (auto-approves, workspace-write sandbox) for round-table; `--dangerously-bypass-approvals-and-sandbox` for review-pr
- **Requires git:** Yes — fails with "Not inside a trusted directory" if not in a git repo. Use `-C {git_dir}` flag (before `exec`) to specify the git directory, or create a temp repo.

#### Commands

**Fresh session:**
```bash
codex exec -C {git_dir} {approval_flag} "$(cat <<'PROMPT_EOF'
{prompt}
PROMPT_EOF
)" 2>&1 | sed 's/\x1b\[[0-9;]*m//g'
```

**Session resume (round-table only):**
```bash
cd {git_dir} && codex exec resume {session_id} "$(cat <<'PROMPT_EOF'
{prompt}
PROMPT_EOF
)" 2>&1 | sed 's/\x1b\[[0-9;]*m//g'
```

**One-shot (review-pr, with inline diff):**
```bash
codex exec --dangerously-bypass-approvals-and-sandbox \
  "$(printf '{prompt_prefix}'; cat {diff_file}; printf '{prompt_suffix}')"
```

#### Prompt Passing
- Use heredoc pattern `"$(cat <<'PROMPT_EOF' ... PROMPT_EOF)"` for multi-line prompts.
- For review-pr: embed diff inline via `printf` + `cat` — Codex cannot reliably read files.

#### Session Resume
- Use `codex exec resume {session_id} "<prompt>"` — NOT `--last`. The `--last` flag cannot be combined with a prompt positional argument.
- **Capture session ID** from round 1 output: look for `session id: {uuid}` in the output header.
- Run from the git directory with `cd {git_dir} &&` for resume commands.
- **Fallback:** If resume fails, fall back to fresh session with full context summary.

#### Output Cleanup
- Remove everything before and including the `codex` marker line.
- Remove the trailing `tokens used` line.
- Strip metadata headers (version, workdir, model, session id).

#### Known Quirks
- Requires a git repository — create temp repo at `/tmp/round-table-workspace` if needed.
- Output contains metadata headers and a `thinking` block before the actual response.

---

### Gemini (Google)

- **Binary:** `gemini`
- **Install:** `npm install -g @anthropic-ai/gemini-cli`
- **Approval mode:** `-y` (yolo mode, auto-approves all tool actions) for round-table; `--yolo` for review-pr
- **Requires git:** No

#### Commands

**Fresh session:**
```bash
gemini -p "$(cat <<'PROMPT_EOF'
{prompt}
PROMPT_EOF
)" -y 2>&1 | sed 's/\x1b\[[0-9;]*m//g'
```

**Session resume (round-table only):**
```bash
gemini -p "$(cat <<'PROMPT_EOF'
{prompt}
PROMPT_EOF
)" -y --resume latest 2>&1 | sed 's/\x1b\[[0-9;]*m//g'
```

**One-shot (review-pr):**
```bash
gemini --yolo -p "{prompt}"
```

#### Prompt Passing
- Use heredoc pattern `"$(cat <<'PROMPT_EOF' ... PROMPT_EOF)"` — works reliably.
- Can reference files directly in prompts (unlike Codex).

#### Session Resume
- `--resume latest` works reliably.
- **Fallback:** If resume fails, fall back to fresh session with full context summary.

#### Output Cleanup
- Remove any `Error executing tool` lines (Gemini internal errors, not actual failures).
- Remove `Loaded cached credentials.` line.

#### Known Quirks
- **Omit `--approval-mode plan`** — requires `experimental.plan` to be enabled and prints a noisy warning.
- May emit harmless internal tool errors (e.g., `Error executing tool run_shell_command: Tool "run_shell_command" not found.`). The actual response text is still valid.

---

### OpenCode

- **Binary:** `opencode`
- **Install:** See [opencode-ai/opencode](https://github.com/opencode-ai/opencode) repo
- **Approval mode:** None available — already runs unrestricted.
- **Model flag:** `-m {model}` to specify model (e.g., `bailian-coding-plan/glm-5`)
- **Requires git:** No

#### Commands

**Fresh session:**
```bash
opencode run {model_flag} "{prompt_flattened}" 2>&1 | sed 's/\x1b\[[0-9;]*m//g'
```

**Session resume (round-table only):**
```bash
opencode run --continue {model_flag} "{prompt_flattened}" 2>&1 | sed 's/\x1b\[[0-9;]*m//g'
```

**One-shot (review-pr):**
```bash
opencode run -m {model} "{prompt}"
```

Where `{model_flag}` is `-m {model}` if a model is configured, or empty string if no model specified.

#### Prompt Passing
- **Do NOT use heredoc patterns.** Pass the prompt as a direct quoted positional argument.
- Heredoc + pipe patterns can cause opencode to hang indefinitely with no output.
- Replace `--` with single dash or em-dash in prompts to avoid flag parsing issues.
- `{prompt_flattened}` means the prompt text with newlines preserved but passed as a single quoted argument.

#### Session Resume
- `--continue` flag, placed before the message: `opencode run --continue "prompt"`.
- **Fallback:** If `--continue` fails, fall back to fresh session with full context summary.

#### Output Cleanup
- Remove the `> build · {model}` header line.

#### Known Quirks
- May run longer than other CLIs. If the Bash tool auto-backgrounds it, use `TaskOutput` to wait for completion (up to 120s), then `TaskStop` if it exceeds the timeout.

---

## Common Conventions

All agent commands follow these conventions:

| Convention | Detail |
|---|---|
| ANSI stripping | Pipe through `sed 's/\x1b\[[0-9;]*m//g'` |
| Timeout | 120-second timeout via Bash tool's `timeout` parameter (do NOT use the `timeout` shell command — unavailable on macOS) |
| Stderr capture | Append `2>&1` to capture both stdout and stderr |
| Error handling | If a CLI is not found (`which` fails), skip it and continue with others |
| Session fallback | If resume/continue fails, fall back to fresh session with full context summary |

---

## Adding a New CLI

To add support for a new external CLI agent:

1. Add a new section under **Supported Agents** above with:
   - Binary name, install command, approval mode, git requirements
   - Fresh command template, session resume command, one-shot command
   - Prompt passing strategy (heredoc vs direct string)
   - Output cleanup rules
   - Known quirks
2. No changes needed to the skill files — they dynamically read this catalog and build commands for each agent in the resolved roster.
