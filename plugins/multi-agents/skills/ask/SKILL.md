---
name: ask
description: Ask all configured AI agents a question in parallel and display their responses. Use when the user says "ask all agents", "ask everyone", or wants to get multiple AI perspectives on a question.
user-invocable: true
version: 0.1.0
---

# Multi-Agent Ask

Send a question to all configured agents in parallel, display each response, and provide an optional synthesis.

## Usage

```
/ask <question>
```

## Instructions

### Step 0: Agent Resolution

1. Read CLAUDE.md files (project-level, then user-level) and look for a `## Multi-Agents` section. Parse each `- {cli-name}` or `- {cli-name}: {model}` line into a roster entry.

2. If no `## Multi-Agents` section is found, auto-detect:
   ```bash
   which codex gemini opencode 2>&1
   ```
   Add one default entry (no model) for each CLI found on `$PATH`.

3. Read `references/agent-catalog.md` (from the plugin's own references directory) to load command templates and CLI-specific details for each agent in the roster.

4. Build the roster with display names per the catalog's display name rules.

### Step 1: Verify CLIs

Run a single parallel setup call:
- `which {all_cli_binaries_in_roster} 2>&1` — verify CLIs
- `git rev-parse --git-dir 2>/dev/null && echo IS_GIT || (mkdir -p /tmp/multi-agents-workspace && cd /tmp/multi-agents-workspace && git init -q 2>/dev/null && echo CREATED_GIT)` — check/create git dir (needed for agents that require git)

Store `git_dir` (cwd if git repo, else `/tmp/multi-agents-workspace`).

### Step 2: Dispatch All Agents in Parallel

Issue all N Bash tool calls in a **single message** so they run concurrently.

For each agent in the roster, build the command using the agent's **fresh session** command template from the catalog:

- **Codex**: Use `--full-auto` approval flag. Use heredoc prompt passing. Pass `-C {git_dir}`.
- **Gemini**: Use `-y` approval flag. Use heredoc prompt passing.
- **OpenCode**: Use direct quoted string (no heredoc). Include `-m {model}` flag if a model is configured.

**Prompt** (adapt per agent's prompt passing strategy):

```
{user_question}
```

Pass the user's question directly — do not wrap it in additional instructions or framing. The user's exact words are the prompt.

**Command conventions** (apply to all agents):
- Pipe through `sed 's/\x1b\[[0-9;]*m//g'` to strip ANSI escape codes
- Use 120-second timeout via the Bash tool's `timeout` parameter
- Append `2>&1` to capture both stdout and stderr

### Step 3: Collect and Display Responses

After all agents return:

1. **Clean each response** using the output cleanup rules from the catalog (strip metadata headers, internal errors, etc.)
2. **Display each response** clearly labeled with the agent's display name:

   ```
   ### {display_name}
   {cleaned_response}
   ```

3. If any agent failed or timed out, note it briefly (e.g., "Codex: [TIMEOUT]" or "OpenCode (GLM-5): [ERROR: ...]") and continue with the others.

### Step 4: Synthesis (optional)

After displaying all responses, provide a brief synthesis (3-5 sentences):
- Where agents agree
- Where they diverge
- Key insights worth highlighting

Keep it short — the individual responses are the main value.

## Error Handling

- If a CLI is not found, skip it and continue with others
- If an agent times out (>120s), use `TaskOutput` to wait; if still running, use `TaskStop` and note `[TIMEOUT]`
- If all agents fail, answer the question yourself
- For agent-specific quirks, refer to the catalog
