# Codex (OpenAI)

- **Binary:** `codex`
- **Install:** `npm install -g @openai/codex`
- **Approval mode:** `--full-auto` (auto-approves, workspace-write sandbox) for round-table; `--dangerously-bypass-approvals-and-sandbox` for review-pr
- **Requires git:** Yes — fails with "Not inside a trusted directory" if not in a git repo. Use `-C {git_dir}` flag (before `exec`) to specify the git directory, or create a temp repo.

## Commands

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

## Prompt Passing
- Use heredoc pattern `"$(cat <<'PROMPT_EOF' ... PROMPT_EOF)"` for multi-line prompts.
- For review-pr: embed diff inline via `printf` + `cat` — Codex cannot reliably read files.

## Session Resume
- Use `codex exec resume {session_id} "<prompt>"` — NOT `--last`. The `--last` flag cannot be combined with a prompt positional argument.
- **Capture session ID** from round 1 output: look for `session id: {uuid}` in the output header.
- Run from the git directory with `cd {git_dir} &&` for resume commands.
- **Fallback:** If resume fails, fall back to fresh session with full context summary.

## Output Cleanup
- Remove everything before and including the `codex` marker line.
- Remove the trailing `tokens used` line.
- Strip metadata headers (version, workdir, model, session id).

## Known Quirks
- Requires a git repository — create temp repo at `/tmp/round-table-workspace` if needed.
- Output contains metadata headers and a `thinking` block before the actual response.
