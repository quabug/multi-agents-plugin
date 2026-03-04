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
which codex gemini opencode pi 2>&1
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

### Pi

- **Binary:** `pi`
- **Install:** See [pi](https://github.com/themaximal1st/pi) repo (also available via Homebrew)
- **Approval mode:** None needed — `-p` (print/non-interactive) mode has no interactive prompts.
- **Model flag:** `--model {model}` to specify model (e.g., `openai/gpt-4o`, `sonnet`, `gemini-2.5-pro`). Supports `provider/model` shorthand.
- **Requires git:** No

#### Commands

**Fresh session:**
```bash
pi -p --no-tools "$(cat <<'PROMPT_EOF'
{prompt}
PROMPT_EOF
)" 2>&1 | sed 's/\x1b\[[0-9;]*m//g'
```

**Session resume (round-table only):**
```bash
pi -p --no-tools --continue "$(cat <<'PROMPT_EOF'
{prompt}
PROMPT_EOF
)" 2>&1 | sed 's/\x1b\[[0-9;]*m//g'
```

**One-shot (review-pr):**
```bash
pi -p --no-tools --model {model} "$(cat <<'PROMPT_EOF'
{prompt}
PROMPT_EOF
)"
```

Where `{model}` is specified via `--model {model}` if a model is configured, or omitted to use the default provider.

#### Prompt Passing
- Use heredoc pattern `"$(cat <<'PROMPT_EOF' ... PROMPT_EOF)"` for multi-line prompts.
- Can reference files directly in prompts using `@file` syntax (e.g., `@diff.patch`), but prefer heredoc for consistency with other agents.

#### Session Resume
- `--continue` flag resumes the previous session.
- **Fallback:** If `--continue` fails, fall back to fresh session with full context summary.

#### Output Cleanup
- Output is generally clean in `-p` mode. Strip ANSI codes as with other agents.
- Remove any startup/loading messages if present.

#### Known Quirks
- Uses `--no-tools` flag for review/ask tasks to prevent Pi from trying to execute tools (read, bash, edit, write) when only a text response is needed.
- Default provider is Google (Gemini). Use `--model provider/model` to switch providers (e.g., `--model openai/gpt-4o`, `--model anthropic/claude-sonnet-4-5-20250514`).

---

## Common Conventions

All agent commands follow these conventions:

| Convention | Detail |
|---|---|
| ANSI stripping | Pipe through `sed 's/\x1b\[[0-9;]*m//g'` |
| Timeout | 120-second timeout via Bash tool's `timeout` parameter (do NOT use the `timeout` shell command — unavailable on macOS) |
| Background dispatch | Use `run_in_background: true` on each Bash call when launching multiple agents in parallel |
| Stderr capture | Append `2>&1` to capture both stdout and stderr |
| Error handling | If a CLI is not found (`which` fails), skip it and continue with others |
| Session fallback | If resume/continue fails, fall back to fresh session with full context summary |

### Result Collection

When collecting results from background agent tasks:

- **Collect sequentially**: Call `TaskOutput` one at a time (one per message, NOT multiple in a single parallel message). If one `TaskOutput` call errors, all sibling tool calls in the same message are cancelled — this cascading failure loses all results.
- **Alternative**: Read the output files directly with the Read tool (the file path is returned when the background task is launched).
- **Timeouts**: If an agent exceeds the 120-second timeout, it is automatically terminated. Note it as "timed out" and proceed with the rest. Use `TaskStop` to terminate any agents still running after collection.
- **Subsequent iterations** (for iterative skills like fix-pr): Consider skipping agents that timed out or failed in a previous iteration to avoid wasting time.

### Output Validation

After collecting and cleaning each agent's output, validate that it contains useful content:

- **Useless output**: Some agents may produce no actionable findings — they echo back the input, produce only metadata/thinking blocks, hit token limits before generating findings, or fail silently with a zero exit code. If an agent's output contains no structured findings after applying cleanup rules, discard it and note "{display_name} produced no actionable output."
- **Do NOT treat empty/useless output as "no issues found"**: Only explicit statements like "No critical or major issues found" count as a clean bill of health. Absence of findings in broken output is not evidence of absence of issues.
- **Model name errors**: Some providers (especially OpenCode) have case-sensitive model names. If an agent fails with a "model not found" or "did you mean" error, note the error, skip that agent, and report the misconfigured model name to the user so they can fix their CLAUDE.md config. Do not retry with a guessed name.

### PR Comment Filtering

When fetching existing PR review comments (for skills like review-pr and fix-pr), filter out noise before analysis:

- **Skip CI bot noise**: Comments from bots posting coverage reports, build status, formatting output, or deployment previews (e.g., `github-actions[bot]`, `codecov[bot]`, `netlify[bot]`). These are informational, not actionable code feedback.
- **Skip quota/limit messages**: Bot comments about usage limits, credits, or billing (e.g., "usage limits reached", "credits must be used").
- **Skip auto-generated summaries**: HTML-heavy bot output, badge images, collapsible coverage tables.
- **Keep actionable review feedback**: Comments from human reviewers and code review bots that provide severity-tagged findings, specific code suggestions, or requested changes (e.g., `gemini-code-assist[bot]` with critical/major/minor labels).

---

## PR Review Extension (`gh pr-review`)

A GitHub CLI extension for posting inline threaded review comments on PRs — similar to what Gemini Code Assist posts. Skills detect this extension at startup and use it when available; if absent, they fall back to default behavior.

**Do NOT auto-install this extension.** If unavailable, silently fall back.

### Detection

```bash
gh pr-review --version 2>/dev/null
```

Store the result as a boolean flag `has_pr_review_ext` (true if exit code 0, false otherwise). Also capture the repo identifier for `-R` flags:

```bash
REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner)
```

### Command Reference

#### Start a pending review

```bash
gh pr-review review --start -R ${REPO} <PR_NUMBER>
```

Returns a `REVIEW_ID` (integer) used by subsequent commands.

#### Add an inline comment to a pending review

```bash
gh pr-review review --add-comment --review-id ${REVIEW_ID} \
  --path <file_path> --line <line_number> \
  --body "comment body" \
  -R ${REPO} <PR_NUMBER>
```

- `--path`: file path relative to repo root (must exist in the diff)
- `--line`: line number in the diff (must be a changed or context line)
- Run sequentially — do NOT add comments in parallel

#### Submit a pending review

```bash
gh pr-review review --submit --review-id ${REVIEW_ID} \
  --event COMMENT \
  --body "summary body" \
  -R ${REPO} <PR_NUMBER>
```

- `--event`: one of `COMMENT`, `APPROVE`, `REQUEST_CHANGES`
- The `--body` becomes the top-level review summary

#### View review threads (structured JSON)

```bash
gh pr-review review view -R ${REPO} --pr <PR_NUMBER> --unresolved
```

Returns JSON array of threads with fields: `thread_id`, `path`, `line`, `body`, `is_resolved`.

#### Reply to a review thread

```bash
gh pr-review comments reply <PR_NUMBER> -R ${REPO} \
  --thread-id <THREAD_ID> --body "reply body"
```

#### Resolve a review thread

```bash
gh pr-review threads resolve --thread-id <THREAD_ID> -R ${REPO} <PR_NUMBER>
```

### Error Handling

- If `--start` or `--add-comment` fails, abandon the inline review and fall back to the default `gh pr review --comment` approach.
- If `threads resolve` fails, log and continue — never abort a fix loop over a resolution failure.
- All commands require the repo flag `-R ${REPO}` to avoid ambiguity in detached-HEAD states.

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
3. Non-agent tools (like `gh pr-review`) that support skills but are not review agents go in their own dedicated section (e.g., "PR Review Extension").
