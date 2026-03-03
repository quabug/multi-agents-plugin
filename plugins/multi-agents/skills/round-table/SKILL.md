---
name: round-table
description: Host a multi-agent round-table discussion. Spawns codex, gemini, and opencode as panelists with distinct roles and moderates a structured multi-round debate. Use when the user says "round-table", "multi-AI discussion", or "panel discussion".
user-invocable: true
version: 0.6.0
---

# Round-Table Multi-Agent Discussion

## Usage

```
/round-table <topic>
```

Host a structured multi-round discussion among 4 AI panelists: Codex, Gemini, OpenCode, and yourself (Claude Code). Each is assigned a distinct role. You (Claude Code) are both a panelist — contributing your own opinions from your assigned role — and the moderator who formulates questions, synthesizes responses, and steers the debate. A markdown transcript is saved to the working directory.

## Participants

4 panelists participate: 3 external CLIs + Claude Code (you). Each external CLI runs in permissive/auto-approve mode.

| Participant | Round 1 Command | Round 2+ Command (session resume) |
|---|---|---|
| Codex | `codex exec -C {git_dir} --full-auto "<prompt>"` | `cd {git_dir} && codex exec resume {session_id} "<prompt>"` |
| Gemini | `gemini -p "<prompt>" -y` | `gemini -p "<prompt>" -y --resume latest` |
| OpenCode | `opencode run "<prompt>"` | `opencode run --continue "<prompt>"` |
| Claude (you) | *(no CLI — respond directly in the conversation)* | *(retain full context natively)* |

**Mode rationale:**
- **Codex `--full-auto`**: Auto-approves commands on failure, workspace-write sandbox. Avoids read-only restrictions that can block tool use.
- **Gemini `-y` (yolo)**: Auto-approves all tool actions without prompting. Prevents the CLI from hanging on approval prompts in non-interactive mode.
- **OpenCode**: No approval flags available — already runs unrestricted.
- **Claude (you)**: No CLI invocation needed. You generate your own response directly, wearing your assigned role hat before switching to neutral moderator voice for the synthesis.

### Agent Memory Strategy

- **Round 1**: Start a fresh session for each participant (no resume flags).
- **Round 2+**: Use resume/continue flags so each agent retains the FULL prior conversation. Only send incremental context (latest synthesis + new question). The CLI tool's internal session handles the complete conversation history.

### CLI-Specific Notes (learned from production use)

**Codex:**
- **Requires a git repository.** Will fail with "Not inside a trusted directory" otherwise.
- Round 1: Use `-C {git_dir}` flag (on the `codex` command, before `exec`) to specify the git directory.
- Round 2+: Use `codex exec resume {session_id} "<prompt>"` — NOT `--last`. The `--last` flag cannot be combined with a prompt positional argument. **You must capture the session ID** from the round 1 output (look for `session id: {uuid}` in the output header) and store it for subsequent rounds.
- Run from the git directory with `cd {git_dir} &&` for resume commands.
- **Output contains metadata headers** (version, workdir, model, session id, etc.) and a `thinking` block before the actual response. Extract only the text after the `codex` marker line for the transcript.

**Gemini:**
- **Omit `--approval-mode plan`** — it requires `experimental.plan` to be enabled and prints a noisy warning. Falls back to default anyway.
- Session resume: `--resume latest` works reliably.
- **May emit harmless internal tool errors** (e.g., `Error executing tool run_shell_command: Tool "run_shell_command" not found.`). These are Gemini's internal errors, not failures. The actual response text is still valid — extract it and ignore the error lines.

**OpenCode:**
- **Do NOT use heredoc patterns** for the prompt. Pass the prompt as a direct quoted positional argument: `opencode run "prompt text here"`.
- Heredoc + pipe patterns can cause opencode to hang indefinitely with no output.
- Session resume: `--continue` flag works, placed before the message: `opencode run --continue "prompt"`.
- May run longer than other CLIs. If the Bash tool auto-backgrounds it, use `TaskOutput` to wait for completion (up to 120s), then `TaskStop` if it exceeds the timeout.

### Command Conventions

All commands:
- Pipe through `sed 's/\x1b\[[0-9;]*m//g'` to strip ANSI escape codes
- Use 120-second timeout via the Bash tool's `timeout` parameter (do NOT use the `timeout` shell command — it's unavailable on macOS)
- Use `2>&1` to capture both stdout and stderr

**Prompt passing strategy by CLI:**
- **Codex**: Use heredoc pattern `"$(cat <<'PROMPT_EOF' ... PROMPT_EOF)"` — works reliably
- **Gemini**: Use heredoc pattern `"$(cat <<'PROMPT_EOF' ... PROMPT_EOF)"` — works reliably
- **OpenCode**: Use direct quoted string only. Flatten the prompt to a single argument. Replace `--` with single dash or em-dash in prompts to avoid flag parsing issues.

## Discussion Protocol

### Phase 0 -- Setup

Minimize tool call rounds. Combine independent checks into parallel calls.

1. **Single parallel setup call** — issue these in one message:
   - `which codex gemini opencode 2>&1` — verify CLIs
   - `git rev-parse --git-dir 2>/dev/null && echo IS_GIT || (mkdir -p /tmp/round-table-workspace && cd /tmp/round-table-workspace && git init -q 2>/dev/null && echo CREATED_GIT)` — check/create git dir
   - `date '+%Y-%m-%d-%H%M%S'` — get timestamp
   - Read `references/roles.md` — load role catalog

2. Pick 4 complementary roles — one for each panelist (Codex, Gemini, OpenCode, Claude). Create productive tension for the topic. Never pick 4 roles that would all agree — ensure at least one contrarian or orthogonal perspective.

3. **Create transcript file and ask user** — write transcript header to `round-table-{topic-slug}-{YYYY-MM-DD-HHmmss}.md`, display all 4 participants + roles, and ask via `AskUserQuestion` if ready or want adjustments.

4. Store `git_dir` (cwd if git repo, else `/tmp/round-table-workspace`) and initialize `codex_session_id=""`.

### Phase 1 -- Rounds (repeat)

1. **Formulate round question**: Based on the topic and (for round 2+) the prior discussion, craft a focused question that advances the debate. Each round should explore a different facet or go deeper on contested points.

2. **Dispatch all 3 participants in parallel** — issue all 3 Bash tool calls in a single message so they run concurrently:

   **Round 1 — fresh session (3 parallel Bash calls):**
   ```bash
   # Codex (capture session_id from output)
   codex exec -C {git_dir} --full-auto "$(cat <<'PROMPT_EOF'
   {round_1_prompt}
   PROMPT_EOF
   )" 2>&1 | sed 's/\x1b\[[0-9;]*m//g'
   ```
   ```bash
   # Gemini (yolo mode)
   gemini -p "$(cat <<'PROMPT_EOF'
   {round_1_prompt}
   PROMPT_EOF
   )" -y 2>&1 | sed 's/\x1b\[[0-9;]*m//g'
   ```
   ```bash
   # OpenCode — direct quoted string, no heredoc
   opencode run "{round_1_prompt_flattened}" 2>&1 | sed 's/\x1b\[[0-9;]*m//g'
   ```

   After Codex returns, parse the output for `session id: {uuid}` and store as `codex_session_id`.

   **Round 2+ — session resume (3 parallel Bash calls):**
   ```bash
   # Codex — use captured session_id, run from git dir
   cd {git_dir} && codex exec resume {codex_session_id} "$(cat <<'PROMPT_EOF'
   {round_n_prompt}
   PROMPT_EOF
   )" 2>&1 | sed 's/\x1b\[[0-9;]*m//g'
   ```
   ```bash
   # Gemini — resume latest session (yolo mode)
   gemini -p "$(cat <<'PROMPT_EOF'
   {round_n_prompt}
   PROMPT_EOF
   )" -y --resume latest 2>&1 | sed 's/\x1b\[[0-9;]*m//g'
   ```
   ```bash
   # OpenCode — continue flag before message
   opencode run --continue "{round_n_prompt_flattened}" 2>&1 | sed 's/\x1b\[[0-9;]*m//g'
   ```

3. **Extract clean responses**: Strip CLI metadata from output:
   - **Codex**: Remove everything before and including the `codex` marker line. Also remove the trailing `tokens used` line.
   - **Gemini**: Remove any `Error executing tool` lines and `Loaded cached credentials.` line.
   - **OpenCode**: Remove the `> build · {model}` header line.

4. **Display + append to transcript in one step**: Display each response to the user, then use the Bash tool with heredoc append (`cat >> transcript.md <<'EOF' ... EOF`) to write the entire round's content (all 4 responses + synthesis) to the transcript in a single operation. This avoids multiple read-then-edit cycles.

5. **Handle errors gracefully** (see Error Handling table below). If a participant fails, skip them and continue with the others. Record the failure in the transcript.

6. **Write Claude's own response**: After reading all 3 external responses, write your own substantive response (200-400 words) from your assigned role's perspective. Engage with and reference the other panelists' points — agree, challenge, or build on them. Display it clearly labeled as "Claude ({your_role})".

7. **Write moderator synthesis**: Switch to neutral moderator voice. Write a synthesis covering all 4 panelists' views (including your own):
   - Key agreements
   - Key disagreements / tensions
   - Novel insights or surprising points
   - Questions to explore in the next round

   Display the synthesis to the user. **Important**: Be fair and even-handed in the synthesis — do not favor your own panelist opinion over others.

8. **Ask the user** via `AskUserQuestion` with these options:
   - **Continue** — proceed to the next round (host picks the next question)
   - **Steer** — user provides direction for the next round's focus
   - **Stop** — end the discussion and move to wrap-up

### Phase 2 -- Wrap-Up

1. Write a final summary section:
   - **Key Takeaways**: The most important insights from the discussion
   - **Points of Consensus**: Where panelists agreed
   - **Unresolved Debates**: Open questions and disagreements
   - **Recommended Next Steps**: Actionable follow-ups

2. Append the summary to the transcript file using `cat >>`.
3. Display the summary to the user and show the transcript file path.

## Prompt Templates

### Round 1 Prompt

```
You are participating in a round-table discussion with 4 panelists.

TOPIC: {topic}

OTHER PANELISTS:
- Codex: {codex_role}
- Gemini: {gemini_role}
- OpenCode: {opencode_role}
- Claude: {claude_role}

YOUR ROLE: {role_name}
{role_description}

CURRENT QUESTION (Round 1):
{round_question}

INSTRUCTIONS:
- Respond from your assigned perspective (200-400 words)
- Be specific and substantive
- Do not simulate other participants or moderator
- Do not include greetings or meta-commentary about being an AI
```

### Round 2+ Prompt

```
ROUND {N} UPDATE:

MODERATOR SYNTHESIS FROM ROUND {N-1}:
{synthesis}

{user_steering_if_any}

CURRENT QUESTION (Round {N}):
{round_question}

Respond from your role as {role_name}. Reference or challenge prior points when relevant. (200-400 words)
```

Note: Round 2+ prompts are intentionally shorter because the CLI tool's session already contains the full conversation history from prior rounds.

## Error Handling

| Scenario | Action |
|---|---|
| CLI not found (`which` fails) | Skip participant, warn user, continue with others |
| Codex "not a trusted directory" | Create temp git repo at `/tmp/round-table-workspace`, retry with `-C` flag |
| Timeout (>120s) or Bash tool auto-backgrounds | Use `TaskOutput` to wait; if still running after 120s, use `TaskStop` and record `[TIMEOUT]` in transcript |
| Non-zero exit / empty output | Capture stderr, record `[ERROR: ...]` in transcript, continue |
| All 3 fail in a round | Alert user via `AskUserQuestion`, offer retry or end discussion |
| Codex session resume fails | Fall back to fresh session (`codex exec -C {git_dir} --full-auto`) with full context summary |
| Gemini `--resume latest` fails | Fall back to fresh session (`gemini -p ... -y`) with full context summary |
| OpenCode `--continue` fails | Fall back to fresh session (`opencode run`) with full context summary |
| Gemini emits internal tool errors | Ignore error lines, extract the actual response text that follows |

## Transcript Format

Write the transcript file using this exact structure:

```markdown
# Round Table: {Topic}

**Date:** {YYYY-MM-DD HH:mm}
**Topic:** {topic}

## Participants

| Participant | Role | Tool |
|---|---|---|
| Codex | {role} | codex |
| Gemini | {role} | gemini |
| OpenCode | {role} | opencode |
| Claude | {role} | claude-code (host) |

---

## Round 1: {Question}

### Codex ({role})
{response}

### Gemini ({role})
{response}

### OpenCode ({role})
{response}

### Claude ({role})
{response}

### Moderator Synthesis
{synthesis}

---

## Round N: {Question}
(same structure per round)

---

## Final Summary

### Key Takeaways
{takeaways}

### Points of Consensus
{consensus}

### Unresolved Debates
{debates}

### Recommended Next Steps
{next_steps}
```

## Important Notes

- **Do NOT commit or push** anything automatically. The transcript file is written to disk but git operations require explicit user request.
- **Parallel dispatch**: Issue all 3 Bash tool calls in a single message so participants run concurrently. This significantly reduces round latency.
- **Minimize tool call rounds**: Combine independent setup checks into parallel calls. Use `cat >>` to append to the transcript instead of read-then-edit cycles. Each round should ideally be: 1 message to dispatch (3 parallel Bash calls) → 1 message to display results + write Claude's response + append transcript + show synthesis + ask user.
- **Role selection**: Always read `references/roles.md` to pick 4 roles. Choose roles that create productive tension — at least one should be likely to disagree with the others on the given topic.
- **Dual hat — panelist then moderator**: In each round, first write your own opinion from your assigned role (be substantive and opinionated), then switch to neutral moderator voice for the synthesis. Keep these clearly separated.
- **Keep moderator synthesis neutral**: In the synthesis section, treat your own panelist response the same as the others. Don't favor your own opinion over theirs.
- **Respect the user**: The user controls pacing. Always ask before continuing to the next round.
- **Clean up temp workspace**: The `/tmp/round-table-workspace` git repo is ephemeral and can be reused across sessions.
