---
name: ask
description: Ask configured AI agents a question in parallel and display their responses. Optionally target a specific agent or model. Use when the user says "ask all agents", "ask codex", "ask gemini", "ask glm-5", or wants AI perspectives on a question.
user-invocable: true
version: 0.8.0
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

First, extract any `--skip {name}` flags from the raw arguments and set them aside for Agent Resolution. Remove them from the argument string before proceeding.

Then parse the remaining argument to determine whether a specific agent is targeted:

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

Follow the procedure in `references/agent-resolution.md` to build the agent roster.

**Apply target filter** (from argument parsing above): If a target was specified, narrow the roster to only the matched agent(s).

### Step 1: Setup

Check/create a git dir for agents that require one (e.g., Codex):
```bash
git rev-parse --git-dir 2>/dev/null && echo IS_GIT || (mkdir -p /tmp/multi-agents-workspace && cd /tmp/multi-agents-workspace && git init -q 2>/dev/null && echo CREATED_GIT)
```

Store `git_dir` (cwd if git repo, else `/tmp/multi-agents-workspace`).

### Step 2: Dispatch Agents in Parallel

For each agent in the (filtered) roster, build the command using the agent's **fresh session** command template from its reference file (`references/{cli-name}.md`). For agents with a model override, include the model flag. Apply all **Common Conventions** from the catalog (ANSI stripping, 600-second timeout, stderr capture, `run_in_background: true`).

**Prompt**: Pass the user's question directly — do not wrap it in additional instructions or framing. The user's exact words are the prompt. Adapt the prompt passing strategy (heredoc vs direct quoted string) per each agent's reference file.

Issue all N Bash tool calls in a **single message** with `run_in_background: true` so they run concurrently (or just 1 foreground call if targeting a single agent).

### Step 3: Collect and Display Responses

Collect results following the **Result Collection** rules from the catalog: call `TaskOutput` sequentially (one per message, never multiple in parallel) to avoid cascading failures. Apply **Output Validation** rules to discard useless output.

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
