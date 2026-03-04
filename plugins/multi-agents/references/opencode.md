# OpenCode

- **Binary:** `opencode`
- **Install:** See [opencode-ai/opencode](https://github.com/opencode-ai/opencode) repo
- **Approval mode:** None available — already runs unrestricted.
- **Model flag:** `-m {model}` to specify model (e.g., `bailian-coding-plan/glm-5`)
- **Requires git:** No

## Commands

**Fresh session:**
```bash
opencode run {model_flag} "{prompt_flattened}" 2>&1 | sed 's/\x1b\[[0-9;]*m//g'
```

**Session resume (round-table only):**
```bash
opencode run --session {session_id} {model_flag} "{prompt_flattened}" 2>&1 | sed 's/\x1b\[[0-9;]*m//g'
```

**One-shot (review-pr):**
```bash
opencode run -m {model} "{prompt}"
```

Where `{model_flag}` is `-m {model}` if a model is configured, or empty string if no model specified.

## Prompt Passing
- **Do NOT use heredoc patterns.** Pass the prompt as a direct quoted positional argument.
- Heredoc + pipe patterns can cause opencode to hang indefinitely with no output.
- Replace `--` with single dash or em-dash in prompts to avoid flag parsing issues.
- `{prompt_flattened}` means the prompt text with newlines preserved but passed as a single quoted argument.

## Session Resume
- **Do NOT use `--continue`.** It resumes the most recent session, which causes cross-contamination when multiple OpenCode agents run in parallel. Additionally, parallel `--continue` can cause `"database is locked"` errors (SQLite contention).
- **Use `--session {session_id}`** in Round 2+ to resume a specific session. After Round 1 completes, capture each agent's session ID by running `opencode session list` and matching the most recent sessions (sorted by Updated time) to the agents that just ran. Session IDs have the format `ses_...`.
- **Fallback:** If `--session` fails, fall back to fresh session with full context summary.

## Output Cleanup
- Remove the `> build · {model}` header line.

## Known Quirks
- May run longer than other CLIs. If the Bash tool auto-backgrounds it, use `TaskOutput` to wait for completion (up to 120s), then `TaskStop` if it exceeds the timeout.
