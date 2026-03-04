---
name: ask
description: Ask configured AI agents a question in parallel and display their responses. Optionally target a specific agent or model. Use when the user says "ask all agents", "ask codex", "ask gemini", "ask glm-5", or wants AI perspectives on a question.
user-invocable: true
version: 0.3.0
---

# Multi-Agent Ask

Send a question to configured agents in parallel (or a specific one), display each response, and provide an optional synthesis.

## Usage

```
/ask <question>                        # ask all configured agents
/ask codex <question>                  # ask a specific CLI agent
/ask gemini <question>                 # ask Gemini only
/ask glm-5 <question>                 # ask by model name (matches OpenCode GLM-5)
/ask opencode:kimi-k2.5 <question>    # ask by cli:model
```

## Argument Parsing

Parse the user's argument to determine whether a specific agent is targeted:

1. **Extract the first word** of the argument (before the question).
2. **Match it against the resolved roster** using these rules (case-insensitive):
   - **Exact CLI name**: `codex`, `gemini`, `opencode` — matches entries with that CLI. If multiple entries share the CLI (e.g., multiple opencode models), ask all of them.
   - **Model name or suffix**: e.g., `glm-5`, `kimi-k2.5`, `minimax-m2.5`, `qwen3.5-plus` — matches the last segment of the model path. Matches a single entry.
   - **cli:model pattern**: e.g., `opencode:glm-5` — matches a specific CLI + model combination.
   - **Display name**: e.g., `GLM-5`, `Kimi-K2.5` — case-insensitive match against display names.
3. **If a match is found**: the first word is the target filter; the rest is the question. Only dispatch to the matched agent(s).
4. **If no match**: the entire argument (including the first word) is the question. Dispatch to all agents.

## Instructions

### Step 0: Agent Resolution

1. Read CLAUDE.md files (project-level, then user-level) and look for a `## Multi-Agents` section. Parse each `- {cli-name}` or `- {cli-name}: {model}` line into a roster entry.

2. If no `## Multi-Agents` section is found, auto-detect:
   ```bash
   which codex gemini opencode pi qwen 2>&1
   ```
   Add one default entry (no model) for each CLI found on `$PATH`.

3. Read `references/agent-catalog.md` for shared conventions and display name rules. Then, for each agent in the roster, read `references/{cli-name}.md` to load that agent's command templates, prompt passing strategy, session resume details, output cleanup rules, and known quirks.

4. Build the roster with display names per the catalog's display name rules.

5. **Apply target filter** (from argument parsing above). If a target was specified, narrow the roster to only the matched agent(s).

### Step 1: Verify CLIs

Run a single parallel setup call:
- `which {all_cli_binaries_in_roster} 2>&1` — verify CLIs
- `git rev-parse --git-dir 2>/dev/null && echo IS_GIT || (mkdir -p /tmp/multi-agents-workspace && cd /tmp/multi-agents-workspace && git init -q 2>/dev/null && echo CREATED_GIT)` — check/create git dir (needed for agents that require git)

Store `git_dir` (cwd if git repo, else `/tmp/multi-agents-workspace`).

### Step 2: Dispatch Agents in Parallel

Issue all N Bash tool calls in a **single message** so they run concurrently (or just 1 call if targeting a single agent).

For each agent in the (filtered) roster, build the command using the agent's **fresh session** command template from the catalog:

- **Codex**: Use `--full-auto` approval flag. Use heredoc prompt passing. Pass `-C {git_dir}`.
- **Gemini**: Use `-y` approval flag. Use heredoc prompt passing.
- **OpenCode**: Use direct quoted string (no heredoc). Include `-m {model}` flag if a model is configured.
- **Pi**: Use `-p --no-tools --no-session` flags. Use direct quoted string (no heredoc). Include `--model {model}` if a model is configured.
- **Qwen**: Use `-y` approval flag. Use heredoc prompt passing. Include `-m {model}` flag if a model is configured.

**Prompt** (adapt per agent's prompt passing strategy):

```
{user_question}
```

Pass the user's question directly — do not wrap it in additional instructions or framing. The user's exact words are the prompt.

**Command conventions** (apply to all agents):
- Pipe through `sed 's/\x1b\[[0-9;]*m//g'` to strip ANSI escape codes
- Use 600-second (10 minute) timeout via the Bash tool's `timeout` parameter
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

4. **Add your own answer** — answer the user's question yourself as an additional perspective. Display it labeled as "Claude Code":

   ```
   ### Claude Code
   {your_answer}
   ```

   Give your honest, independent opinion. Don't just echo or summarize what agents said — contribute your own analysis, knowledge, and perspective on the question.

### Step 4: Synthesis

- **Multiple agents**: Provide a brief synthesis (3-5 sentences) — where agents (including yourself) agree, diverge, and key insights.
- **Single agent**: Provide a brief comparison between the agent's response and your own — note agreements and differences.

## Error Handling

- If a CLI is not found, skip it and continue with others
- If an agent times out (>600s), use `TaskOutput` to wait; if still running, use `TaskStop` and note `[TIMEOUT]`
- If the targeted agent is not found in the roster, list available agents and ask the user to pick one
- If all agents fail, answer the question yourself
- For agent-specific quirks, refer to the catalog
