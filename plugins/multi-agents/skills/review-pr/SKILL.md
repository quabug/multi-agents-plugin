---
name: review-pr
description: This skill should be used when the user asks to "review a PR", "review pull request", "PR review", or wants a multi-agent code review of a GitHub pull request using configurable AI tools in parallel.
version: 0.8.0
---

# Multi-Agent PR Review

Review a GitHub PR using multiple AI agents in parallel, then synthesize their findings into a single review comment.

## Arguments

The user provides a PR number or URL (e.g., `42` or `https://github.com/owner/repo/pull/42`). Extract the PR number from the argument. Also extract any `--skip {name}` flags and pass them to Agent Resolution for exclusion.

## Instructions

### Step 0: Agent Resolution

Follow the procedure in `references/agent-resolution.md` to build the agent roster.

### Step 1: Fetch PR Context

Capture the repo identifier and detect the `gh pr-review` extension first — `REPO` is used by subsequent commands:

```bash
REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner)
gh pr-review --version 2>/dev/null
```

Store `has_pr_review_ext` (true if exit code 0, false otherwise) and `REPO`. Refer to the **PR Review Extension** section in `references/agent-catalog.md` for the full command reference.

Fetch the PR diff and metadata:

```bash
mkdir -p pr-reviews
gh pr diff <PR_NUMBER> > pr-reviews/pr<PR_NUMBER>.diff
gh pr view <PR_NUMBER> --json title,body,baseRefName,headRefName
```

Fetch existing review comments and conversations so agents can consider prior feedback:

```bash
gh api repos/${REPO}/pulls/<PR_NUMBER>/comments --jq '.[] | "[\(.user.login)] \(.path):\(.line // .original_line) — \(.body)"' > pr-reviews/pr<PR_NUMBER>.comments 2>/dev/null
gh api repos/${REPO}/pulls/<PR_NUMBER>/reviews --jq '.[] | select(.body != "") | "[\(.user.login)] (\(.state)) — \(.body)"' >> pr-reviews/pr<PR_NUMBER>.comments 2>/dev/null
gh api repos/${REPO}/issues/<PR_NUMBER>/comments --jq '.[] | "[\(.user.login)] — \(.body)"' >> pr-reviews/pr<PR_NUMBER>.comments 2>/dev/null
```

Apply the **PR Comment Filtering** rules from the catalog to skip CI bot noise, quota messages, and auto-generated summaries. Read both the diff and comments files to understand the scope, context, and any prior feedback.

**Large diff guard**: Check `wc -l pr-reviews/pr<PR_NUMBER>.diff`. If the diff exceeds 5000 lines, warn the user — agents (especially Codex, which embeds diffs inline) may hit token limits or produce lower-quality reviews on very large diffs. Ask whether to proceed or focus on specific files.

### Step 2: Launch Review Agents in Parallel

Issue all N Bash tool calls in a **single message** with `run_in_background: true` so they run concurrently. Each agent reviews the diff from a different perspective.

For each agent in the roster, build the review command using the agent's **one-shot** command template from its reference file (`references/{cli-name}.md`). Apply all **Common Conventions** from the catalog (ANSI stripping, 600-second timeout, stderr capture).

- **Agents that cannot reliably read files** (e.g., Codex per the catalog): Embed the diff AND existing comments inline in the prompt via `printf` + `cat`.
- **Agents that can reference files** (e.g., Gemini, OpenCode per the catalog): Reference the diff and comments files by path in the prompt.
- **Agents with a model override**: Include the model flag (e.g., `-m {model}` for OpenCode).

**Review prompt** (adapt per agent's prompt passing strategy from the catalog):

```
Review the PR diff for correctness, bugs, security issues, architecture patterns, performance, and code quality.
Focus on logic errors, edge cases, and best practices.
Consider the existing review comments below — do not duplicate issues already raised, and note if prior feedback has been addressed or not.
Output your findings as a structured list with severity labels (critical/major/minor).

--- DIFF ---
{diff content or file reference}

--- EXISTING REVIEW COMMENTS ---
{comments content or file reference}
```

### Step 3: Wait for ALL Agents, Then Synthesize

**IMPORTANT**: Wait for ALL launched review tools to complete before posting the review. Do NOT post with partial results.

Collect results following the **Result Collection** and **Output Validation** rules from the catalog: call `TaskOutput` sequentially (one per message, never multiple in parallel) to avoid cascading failures. If a tool exceeds the 600-second timeout, note it as "did not complete" and proceed with the rest.

After all tools finish:

1. **Clean** findings from each tool. Apply output cleanup rules from the catalog. Discard useless output per the Output Validation rules.
2. **Cross-reference with existing comments**: skip issues already raised in prior reviews; note if prior feedback has been addressed by the current diff
3. **Categorize** new issues by severity:
   - **Critical**: bugs, security issues, data corruption risks
   - **Major**: logic errors, performance problems, API misuse
   - **Minor**: style, naming, documentation
4. **Identify consensus**: issues flagged by 2+ tools are high-confidence
5. **Dismiss false positives**: if a finding is clearly wrong given project context, dismiss it with reasoning
6. **Add your own independent review**: Review the diff yourself as an additional reviewer. Identify issues the agents may have missed, validate or challenge their findings, and contribute your own perspective on correctness, architecture, and edge cases. Your findings are treated on equal footing with agent findings — label them as sourced by "Claude Code" in the attribution table
7. **Classify findings for inline posting** (only when `has_pr_review_ext` is true):
   - Parse the diff to build a map of `{file → [changed line numbers]}`
   - For each synthesized finding, check if it has a specific `{path, line}` reference:
     - **Inline finding**: The cited `path` exists in the diff AND the cited `line` is within ±5 lines of a changed line → will be posted as an inline comment
     - **General finding**: No specific location, or the cited location doesn't match the diff → goes in the review body summary
   - This classification is only used for Step 4a; Step 4b ignores it

### Step 4: Post Review

Use **Step 4a** when `has_pr_review_ext` is true. If 4a fails at any point (start or add-comment errors), abandon and fall back to **Step 4b**. Use **Step 4b** directly when `has_pr_review_ext` is false.

#### Step 4a: Inline Threaded Review (when `has_pr_review_ext` is true)

Post inline comments on specific file+line locations, with a summary body for general findings.

1. **Start a pending review:**
   ```bash
   gh pr-review review --start -R ${REPO} <PR_NUMBER>
   ```
   Capture the `REVIEW_ID` from the output. If this fails, fall back to Step 4b.

2. **Add inline comments** — for each **inline finding** (has valid `{path, line}` in the diff), run sequentially (NOT in parallel):
   ```bash
   gh pr-review review --add-comment --review-id ${REVIEW_ID} \
     --path <file_path> --line <line_number> \
     --body "[severity] finding description (sourced by: agent1, agent2)" \
     -R ${REPO} <PR_NUMBER>
   ```
   If any `--add-comment` fails, abandon the pending review and fall back to Step 4b.

3. **Submit the review** with a summary body containing general findings and the attribution table:
   ```bash
   gh pr-review review --submit --review-id ${REVIEW_ID} \
     --event COMMENT \
     --body "$(cat <<'EOF'
   ## Multi-Agent PR Review

   ### Summary
   [1-2 sentence overall assessment]

   ### General Findings
   [Findings without specific file+line locations, or that couldn't be mapped to the diff]

   ### Suggestions
   [Non-blocking improvements worth considering]

   ### Notes
   [Any dismissed false positives or context-specific observations]

   ### Agent Attribution
   | Finding | {display_name_1} | {display_name_2} | ... |
   |---------|---|---|---|
   | [issue] | x |   | x |

   ---
   ### Claude Code's Own Findings
   [Your independent findings that agents missed or that you want to highlight — if none beyond what agents found, state "No additional findings."]

   ---
   Reviewed by: Claude Code + {display_name_1} + {display_name_2} + ...
   EOF
   )" \
     -R ${REPO} <PR_NUMBER>
   ```

#### Step 4b: Fallback — Single Review Comment (when `has_pr_review_ext` is false, or 4a failed)

Post a single synthesized review comment on the PR via `gh pr review`. Build the attribution table dynamically from the roster:

```bash
gh pr review <PR_NUMBER> --comment --body "$(cat <<'EOF'
## Multi-Agent PR Review

### Summary
[1-2 sentence overall assessment]

### Critical Issues
[Issues flagged by multiple agents or clearly important — if none, state "No critical issues found."]

### Suggestions
[Non-blocking improvements worth considering]

### Notes
[Any dismissed false positives or context-specific observations]

### Claude Code's Own Findings
[Your independent findings that agents missed or that you want to highlight — if none beyond what agents found, state "No additional findings."]

### Agent Attribution
| Finding | Claude Code | {display_name_1} | {display_name_2} | ... |
|---------|---|---|---|---|
| [issue] | x | x |   | x |

---
Reviewed by: Claude Code + {display_name_1} + {display_name_2} + ...
EOF
)"
```

### Step 5: Cleanup

```bash
rm -f pr-reviews/pr<PR_NUMBER>.diff pr-reviews/pr<PR_NUMBER>.comments
rmdir pr-reviews 2>/dev/null
```

## Error Handling

- If a tool is not installed, skip it and note it in the review
- If `gh` is not authenticated, prompt the user to run `gh auth login`
- If the PR number is invalid, report the error clearly
- If all external tools fail, provide your own review based on the diff
- For agent-specific error handling, refer to the catalog's known quirks and output cleanup rules
- If `gh pr-review` extension commands fail during Step 4a, abandon the inline review and fall back to Step 4b (single comment)
- If `gh pr-review` is installed but returns errors for a specific PR (e.g., permissions), fall back to Step 4b silently
