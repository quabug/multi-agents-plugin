# Gemini (Google)

- **Binary:** `gemini`
- **Permissions:** Follow [shared conventions](agent-catalog.md); use current CLI help to restrict question/review participants to the permitted task scope.
- **Requires git:** No
- **Model flag:** `-m {model}` when selected, otherwise empty.

## Commands

**Fresh session:**
```bash
gemini {model_flag} -p "$(cat <<'PROMPT_EOF'
{prompt}
PROMPT_EOF
)" {permission_flags} 2>&1
```

**Session resume (round-table only):**
```bash
gemini {model_flag} -p "$(cat <<'PROMPT_EOF'
{prompt}
PROMPT_EOF
)" {permission_flags} --resume {session_uuid} 2>&1
```

**One-shot (review-pr):**
```bash
gemini {permission_flags} {model_flag} -p "{prompt}"
```

## Prompt Passing
- Use heredoc pattern `"$(cat <<'PROMPT_EOF' ... PROMPT_EOF)"` — works reliably.
- Can reference accessible files directly in prompts.

## Session Resume
- **Do NOT use `--resume latest`.** It resumes the most recent session, which causes cross-contamination when multiple Gemini agents run in parallel.
- **Use `--resume {uuid}`** in Round 2+ to resume a specific session. After Round 1 completes, capture each agent's session UUID by running `gemini --list-sessions` — output shows index, title (contains prompt text), and UUID in brackets. Match sessions to agents by title/prompt content.
- **Fallback:** If resume fails, fall back to fresh session with full context summary.

## Output Cleanup
- Preserve tool errors that limit evidence; separate them from the answer rather than treating them as findings.
- Remove `Loaded cached credentials.` line.

## Known Quirks
- The installed help advertises `--approval-mode plan` for read-only work; verify any version-specific setup before relying on it.
- May emit harmless internal tool errors (e.g., `Error executing tool run_shell_command: Tool "run_shell_command" not found.`). Assess whether those errors affected the evidence before relying on the response.
