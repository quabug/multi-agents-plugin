---
name: round-table
description: Host a multi-agent round-table discussion. Spawns configurable external CLI agents as panelists with distinct roles and moderates a structured multi-round debate. Use when the user says "round-table", "multi-AI discussion", or "panel discussion".
user-invocable: true
version: 0.8.0
---

# Round-Table Multi-Agent Discussion

## Usage

```
/round-table <topic>
/round-table --skip codex <topic>
```

Extract any `--skip {name}` flags from the arguments and pass them to Agent Resolution for exclusion. The remaining argument is the topic.

Host a structured multi-round discussion among N+1 AI panelists: N external CLI agents (configured or auto-detected) plus yourself (Claude Code). Each is assigned a distinct role. You (Claude Code) are both a panelist — contributing your own opinions from your assigned role — and the moderator who formulates questions, synthesizes responses, and steers the debate. A markdown transcript is saved to the working directory.

## Participants

N+1 panelists participate: N external CLIs (from resolved roster) + Claude Code (you). Each external CLI runs in permissive/auto-approve mode.

The roster is determined by the **Agent Resolution** step in Phase 0. Refer to `references/agent-catalog.md` for shared conventions and each per-agent file (`references/{cli-name}.md`) for command templates, flags, prompt passing strategy, output cleanup rules, and known quirks.

## Discussion Protocol

### Phase 0 -- Setup

Minimize tool call rounds. Combine independent checks into parallel calls.

1. **Agent Resolution** — follow the procedure in `references/agent-resolution.md` to build the agent roster.

2. **Single parallel setup call** — issue these in one message:
   - `git rev-parse --git-dir 2>/dev/null && echo IS_GIT || (mkdir -p /tmp/multi-agents-workspace && cd /tmp/multi-agents-workspace && git init -q 2>/dev/null && echo CREATED_GIT)` — check/create git dir (needed for agents that require git)
   - `date '+%Y-%m-%d-%H%M%S'` — get timestamp

3. **Generate N+1 complementary roles** — one for each panelist (N external agents + Claude). Invent roles tailored to the specific topic that create productive tension. Guidelines:
   - Each role has a **name** (e.g., "Pragmatist", "Security Advocate", "Game Designer") and a 1-sentence **focus** describing what that perspective cares about.
   - Pick at least one role likely to disagree with the others — never all roles that would agree.
   - Include an orthogonal voice: one role should bring a perspective the others wouldn't naturally consider.
   - Avoid redundant roles with overlapping concerns.
   - Roles can be technical (Architect, Performance Engineer, DevOps/SRE), general (Skeptic, Optimist, Ethicist, Economist), or domain-specific (Game Designer, Product Manager, Data Scientist) — whatever fits the topic best.

4. **Create transcript file and ask user** — write transcript header to `round-table-{topic-slug}-{YYYY-MM-DD-HHmmss}.md`, display all participants + roles, and ask via `AskUserQuestion` if ready or want adjustments.

5. Store `git_dir` (cwd if git repo, else `/tmp/multi-agents-workspace`) and initialize session state per agent (e.g., `codex_session_id=""`).

### Phase 1 -- Rounds (repeat)

1. **Formulate round question**: Based on the topic and (for round 2+) the prior discussion, craft a focused question that advances the debate. Each round should explore a different facet or go deeper on contested points.

2. **Dispatch all N participants in parallel** — issue all N Bash tool calls in a single message so they run concurrently. For each agent in the roster, build the command from the catalog:

   **Round 1 — fresh session:**
   Use each agent's **fresh session** command template from the catalog. Substitute `{prompt}` with the round 1 prompt (see Prompt Templates below). For agents with model overrides, include the model flag.

   **Round 2+ — session resume:**
   Use each agent's **session resume** command template from the catalog. For agents that don't support session resume, use the fresh session command with a full context summary.

   After each agent returns, perform any agent-specific post-processing:
   - Capture session IDs where needed (e.g., Codex `session id: {uuid}`)
   - Apply output cleanup rules from the catalog

   **Command conventions** (apply to all agents):
   - Pipe through `sed 's/\x1b\[[0-9;]*m//g'` to strip ANSI escape codes
   - Use 600-second (10 minute) timeout via the Bash tool's `timeout` parameter (do NOT use the `timeout` shell command — unavailable on macOS)
   - Append `2>&1` to capture both stdout and stderr

3. **Extract clean responses**: For each agent, apply the output cleanup rules defined in the catalog.

4. **Display + append to transcript in one step**: Display each response to the user, then use the Bash tool with heredoc append (`cat >> transcript.md <<'EOF' ... EOF`) to write the entire round's content (all N+1 responses + synthesis) to the transcript in a single operation.

5. **Handle errors gracefully** (see Error Handling table below). If a participant fails, skip them and continue with the others. Record the failure in the transcript.

6. **Write Claude's own response**: After reading all N external responses, write your own substantive response (200-400 words) from your assigned role's perspective. Engage with and reference the other panelists' points — agree, challenge, or build on them. Display it clearly labeled as "Claude ({your_role})".

7. **Write moderator synthesis**: Switch to neutral moderator voice. Write a synthesis covering all N+1 panelists' views (including your own):
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
You are participating in a round-table discussion with {N+1} panelists.

TOPIC: {topic}

OTHER PANELISTS:
{for each panelist: "- {display_name}: {role_name}"}

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

Note: Round 2+ prompts are intentionally shorter because the CLI tool's session already contains the full conversation history from prior rounds (for agents that support session resume).

### Agent Memory Strategy

- **Round 1**: Start a fresh session for each participant (no resume flags).
- **Round 2+**: Use resume/continue flags (per catalog) so each agent retains the FULL prior conversation. Only send incremental context (latest synthesis + new question). The CLI tool's internal session handles the complete conversation history.

## Error Handling

| Scenario | Action |
|---|---|
| CLI not found (`which` fails) | Skip participant, warn user, continue with others |
| Agent requires git but no git dir | Create temp git repo at `/tmp/multi-agents-workspace`, retry with appropriate flag |
| Timeout (>600s) or Bash tool auto-backgrounds | Use `TaskOutput` to wait; if still running after 600s, use `TaskStop` and record `[TIMEOUT]` in transcript |
| Non-zero exit / empty output | Capture stderr, record `[ERROR: ...]` in transcript, continue |
| All external agents fail in a round | Alert user via `AskUserQuestion`, offer retry or end discussion |
| Session resume fails | Fall back to fresh session with full context summary (per catalog) |
| Agent emits internal tool errors | Ignore error lines per catalog cleanup rules, extract the actual response text |

## Transcript Format

Write the transcript file using this structure. The participant table and per-round sections are dynamic based on the resolved roster:

```markdown
# Round Table: {Topic}

**Date:** {YYYY-MM-DD HH:mm}
**Topic:** {topic}

## Participants

| Participant | Role | Tool |
|---|---|---|
{for each external agent: "| {display_name} | {role} | {cli_name} |"}
| Claude | {role} | claude-code (host) |

---

## Round 1: {Question}

{for each external agent:
### {display_name} ({role})
{response}
}

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
- **Parallel dispatch**: Issue all N Bash tool calls in a single message so participants run concurrently. This significantly reduces round latency.
- **Minimize tool call rounds**: Combine independent setup checks into parallel calls. Use `cat >>` to append to the transcript instead of read-then-edit cycles. Each round should ideally be: 1 message to dispatch (N parallel Bash calls) -> 1 message to display results + write Claude's response + append transcript + show synthesis + ask user.
- **Role generation**: Generate roles tailored to the topic at hand. Choose roles that create productive tension — at least one should be likely to disagree with the others on the given topic.
- **Dual hat — panelist then moderator**: In each round, first write your own opinion from your assigned role (be substantive and opinionated), then switch to neutral moderator voice for the synthesis. Keep these clearly separated.
- **Keep moderator synthesis neutral**: In the synthesis section, treat your own panelist response the same as the others. Don't favor your own opinion over theirs.
- **Respect the user**: The user controls pacing. Always ask before continuing to the next round.
- **Clean up temp workspace**: The `/tmp/multi-agents-workspace` git repo is ephemeral and can be reused across sessions.
- **Catalog is authoritative**: Shared conventions live in `references/agent-catalog.md` and per-agent commands, flags, quirks, and output cleanup rules live in `references/{cli-name}.md`. Do not hardcode CLI details in this file.
