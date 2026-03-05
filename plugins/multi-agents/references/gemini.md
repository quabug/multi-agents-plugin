# Gemini (Google)

- **Binary:** `gemini`
- **Install:** `npm install -g @anthropic-ai/gemini-cli`
- **Approval mode:** `-y` (short for `--yolo`, auto-approves all tool actions)
- **Requires git:** No

## Commands

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
)" -y --resume {session_uuid} 2>&1 | sed 's/\x1b\[[0-9;]*m//g'
```

**One-shot (review-pr):**
```bash
gemini -y -p "{prompt}"
```

## Prompt Passing
- Use heredoc pattern `"$(cat <<'PROMPT_EOF' ... PROMPT_EOF)"` — works reliably.
- Can reference files directly in prompts (unlike Codex).

## Session Resume
- **Do NOT use `--resume latest`.** It resumes the most recent session, which causes cross-contamination when multiple Gemini agents run in parallel.
- **Use `--resume {uuid}`** in Round 2+ to resume a specific session. After Round 1 completes, capture each agent's session UUID by running `gemini --list-sessions` — output shows index, title (contains prompt text), and UUID in brackets. Match sessions to agents by title/prompt content.
- **Fallback:** If resume fails, fall back to fresh session with full context summary.

## Output Cleanup
- Remove any `Error executing tool` lines (Gemini internal errors, not actual failures).
- Remove `Loaded cached credentials.` line.

## Known Quirks
- **Omit `--approval-mode plan`** — requires `experimental.plan` to be enabled and prints a noisy warning.
- May emit harmless internal tool errors (e.g., `Error executing tool run_shell_command: Tool "run_shell_command" not found.`). The actual response text is still valid.
