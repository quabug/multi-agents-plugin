# Codex (OpenAI)

- **Binary:** `codex`
- **Permissions:** Follow [shared conventions](agent-catalog.md); use current CLI help to restrict question/review participants to the permitted task scope.
- **Workspace:** Use the intended repository or an isolated temporary workspace. Check current `codex exec --help` for repository-check options if Git context is unavailable.

## Commands

**Fresh session:**
```bash
codex exec -C {git_dir} {permission_flags} {model_flag} "$(cat <<'PROMPT_EOF'
{prompt}
PROMPT_EOF
)" 2>&1
```

**Session resume (round-table only):**
```bash
cd {git_dir} && codex exec resume {model_flag} {session_id} "$(cat <<'PROMPT_EOF'
{prompt}
PROMPT_EOF
)" 2>&1
```

**One-shot (review-pr, prepared prompt on stdin):**
```bash
codex exec {permission_flags} {model_flag} \
  - < {prompt_file}
```

`{model_flag}` is `--model {model}` when selected, otherwise empty. Substitute properly
quoted arguments or an argument array; capture the process exit status before cleaning output.

## Prompt Passing
- Use heredoc pattern `"$(cat <<'PROMPT_EOF' ... PROMPT_EOF)"` for multi-line prompts.
- For reviews, provide accessible read-only files or a prompt on stdin. Include diff text only when the participant cannot access its source.

## Session Resume
- Use `codex exec resume {session_id} "<prompt>"`; avoid `--last` because parallel participants can resume the wrong session.
- **Capture session ID** from round 1 output: look for `session id: {uuid}` in the output header.
- Run from the git directory with `cd {git_dir} &&` for resume commands.
- **Fallback:** If resume fails, fall back to fresh session with full context summary.

## Output Cleanup
- Remove everything before and including the `codex` marker line.
- Remove the trailing `tokens used` line.
- Strip metadata headers (version, workdir, model, session id).

## Known Quirks
- Use a unique temporary workspace when isolation is needed; do not reuse another task's files.
- Output contains metadata headers and a `thinking` block before the actual response.
