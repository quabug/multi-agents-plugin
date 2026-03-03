---
name: review-pr
description: This skill should be used when the user asks to "review a PR", "review pull request", "PR review", or wants a multi-agent code review of a GitHub pull request using configurable AI tools in parallel.
version: 0.3.0
---

# Multi-Agent PR Review

Review a GitHub PR using multiple AI agents in parallel, then synthesize their findings into a single review comment.

## Arguments

The user provides a PR number or URL (e.g., `42` or `https://github.com/owner/repo/pull/42`). Extract the PR number from the argument.

## Instructions

### Step 0: Agent Resolution

Determine the agent roster before doing anything else.

1. Read CLAUDE.md files (project-level, then user-level) and look for a `## Multi-Agents` section. Parse each `- {cli-name}` or `- {cli-name}: {model}` line into a roster entry.

2. If no `## Multi-Agents` section is found, auto-detect:
   ```bash
   which codex gemini opencode 2>&1
   ```
   Add one default entry (no model) for each CLI found on `$PATH`.

3. Read `references/agent-catalog.md` (from the plugin's own references directory) to load command templates and CLI-specific details for each agent in the roster.

4. Build the roster with display names per the catalog's display name rules.

### Step 1: Fetch PR Context

Fetch the PR diff, metadata, and existing review comments:

```bash
mkdir -p pr-reviews
gh pr diff <PR_NUMBER> > pr-reviews/pr<PR_NUMBER>.diff
gh pr view <PR_NUMBER> --json title,body,baseRefName,headRefName
```

Fetch existing review comments and conversations so agents can consider prior feedback:

```bash
gh api repos/{owner}/{repo}/pulls/<PR_NUMBER>/comments --jq '.[] | "[\(.user.login)] \(.path):\(.line // .original_line) — \(.body)"' > pr-reviews/pr<PR_NUMBER>.comments 2>/dev/null
gh api repos/{owner}/{repo}/pulls/<PR_NUMBER>/reviews --jq '.[] | select(.body != "") | "[\(.user.login)] (\(.state)) — \(.body)"' >> pr-reviews/pr<PR_NUMBER>.comments 2>/dev/null
gh api repos/{owner}/{repo}/issues/<PR_NUMBER>/comments --jq '.[] | "[\(.user.login)] — \(.body)"' >> pr-reviews/pr<PR_NUMBER>.comments 2>/dev/null
```

Read both the diff and comments files to understand the scope, context, and any prior feedback.

### Step 2: Launch Review Agents in Parallel

Run ALL agents from the resolved roster **in parallel** as background bash commands. Each agent reviews the diff from a different perspective.

For each agent in the roster, build the review command using the agent's **one-shot** command template from the catalog:

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

**IMPORTANT**: Wait for ALL launched review tools to complete before posting the review. Do NOT post with partial results. If a tool takes more than 5 minutes with no output progress, note it as "did not complete" and proceed with the rest.

After all tools finish:

1. **Collect** findings from each tool. Apply output cleanup rules from the catalog.
2. **Cross-reference with existing comments**: skip issues already raised in prior reviews; note if prior feedback has been addressed by the current diff
3. **Categorize** new issues by severity:
   - **Critical**: bugs, security issues, data corruption risks
   - **Major**: logic errors, performance problems, API misuse
   - **Minor**: style, naming, documentation
4. **Identify consensus**: issues flagged by 2+ tools are high-confidence
5. **Dismiss false positives**: if a finding is clearly wrong given project context, dismiss it with reasoning
6. **Add your own review insights** based on your reading of the diff, existing comments, and project knowledge

### Step 4: Post Review

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

### Agent Attribution
| Finding | {display_name_1} | {display_name_2} | ... |
|---------|---|---|---|
| [issue] | x |   | x |

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
