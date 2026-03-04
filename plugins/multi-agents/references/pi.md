# Pi

- **Binary:** `pi`
- **Install:** See [pi](https://github.com/themaximal1st/pi) repo (also available via Homebrew)
- **Approval mode:** None needed — `-p` (print/non-interactive) mode has no interactive prompts.
- **Model flag:** `--model {model}` to specify model (e.g., `openai/gpt-4o`, `sonnet`, `gemini-2.5-pro`). Supports `provider/model` shorthand.
- **Requires git:** No

## Commands

**Fresh session:**
```bash
pi -p --no-tools --session-dir {session_dir} {model_flag} "{prompt_flattened}" 2>&1 | sed 's/\x1b\[[0-9;]*m//g'
```

**Session resume (round-table only):**
```bash
pi -p --no-tools --session {session_file} {model_flag} "{prompt_flattened}" 2>&1 | sed 's/\x1b\[[0-9;]*m//g'
```

**One-shot (review-pr):**
```bash
pi -p --no-tools --no-session --model {model} "{prompt_flattened}"
```

Where `{model_flag}` is `--model {model}` if a model is configured, or empty string if no model specified.

## Prompt Passing
- **Do NOT use heredoc patterns.** Pass the prompt as a direct quoted positional argument.
- Heredoc + pipe patterns cause Pi to hang indefinitely with no output in zsh.
- `{prompt_flattened}` means the prompt text with newlines preserved but passed as a single quoted argument.
- Can reference files directly in prompts using `@file` syntax (e.g., `@diff.patch`).

## Session Resume
- **Do NOT use `--continue`.** It resumes the most recently modified session file, which causes cross-contamination when multiple Pi agents run in parallel (all agents resume the same session instead of their own).
- **Use `--session-dir {dir}`** in Round 1 to isolate each agent's session into a unique directory (e.g., `/tmp/round-table-sessions/glm-5/`).
- **Use `--session {path}`** in Round 2+ to resume a specific session file. After Round 1, find the session file: `ls {session_dir}/` — there will be exactly one `.jsonl` file.
- **Use `--no-session`** for one-shot tasks (review-pr, ask) where session history is not needed.
- **Fallback:** If `--session` fails, fall back to fresh session with full context summary.

## Output Cleanup
- Output is generally clean in `-p` mode. Strip ANSI codes as with other agents.
- Remove any startup/loading messages if present.

## Known Quirks
- Uses `--no-tools` flag for review/ask tasks to prevent Pi from trying to execute tools (read, bash, edit, write) when only a text response is needed.
- Default provider is Google (Gemini). Use `--model provider/model` to switch providers (e.g., `--model openai/gpt-4o`, `--model anthropic/claude-sonnet-4-5-20250514`).
