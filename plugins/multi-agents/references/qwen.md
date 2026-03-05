# Qwen (Alibaba)

- **Binary:** `qwen`
- **Install:** `npm install -g @anthropic-ai/qwen-code` or see [Qwen Code](https://github.com/anthropics/qwen-code) repo
- **Approval mode:** `-y` (short for `--yolo`, auto-approves all tool actions)
- **Model flag:** `-m {model}` to specify model
- **Requires git:** No

## Commands

**Fresh session:**
```bash
qwen "$(cat <<'PROMPT_EOF'
{prompt}
PROMPT_EOF
)" -y --session-id {uuid} 2>&1 | sed 's/\x1b\[[0-9;]*m//g'
```

**Session resume (round-table only):**
```bash
qwen "$(cat <<'PROMPT_EOF'
{prompt}
PROMPT_EOF
)" -y --resume {session_uuid} 2>&1 | sed 's/\x1b\[[0-9;]*m//g'
```

**One-shot (review-pr):**
```bash
qwen -y "$(cat <<'PROMPT_EOF'
{prompt}
PROMPT_EOF
)" 2>&1 | sed 's/\x1b\[[0-9;]*m//g'
```

## Prompt Passing
- Use heredoc pattern `"$(cat <<'PROMPT_EOF' ... PROMPT_EOF)"` — works reliably (same as Gemini).
- Can reference files directly in prompts.

## Session Resume
- **Do NOT use `--continue`.** It resumes the most recent session, which causes cross-contamination when multiple Qwen agents run in parallel.
- **Use `--session-id {uuid}`** in Round 1 to pre-assign a UUID (generate with `uuidgen | tr '[:upper:]' '[:lower:]'`). This avoids needing to capture session IDs after the fact.
- **Use `--resume {uuid}`** in Round 2+ to resume a specific session by the same UUID assigned in Round 1.
- **Fallback:** If resume fails, fall back to fresh session with full context summary.

## Output Cleanup
- Output is generally clean. Strip ANSI codes as with other agents.

## Known Quirks
- Very similar to Gemini CLI in flags and behavior.
- `--session-id` requires a valid UUID format (e.g., `123e4567-e89b-12d3-a456-426614174000`).
- Supports multiple auth types: `openai`, `anthropic`, `qwen-oauth`, `gemini`, `vertex-ai`.
