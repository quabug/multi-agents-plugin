---
name: review-pr
description: This skill should be used when the user asks to "review a PR", "review pull request", "PR review", or wants a multi-agent code review of a GitHub pull request using multiple AI tools in parallel.
version: 0.2.0
---

# Multi-Agent PR Review

Review a GitHub PR using multiple AI agents in parallel, then synthesize their findings into a single review comment.

## Arguments

The user provides a PR number or URL (e.g., `42` or `https://github.com/owner/repo/pull/42`). Extract the PR number from the argument.

## Instructions

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

Read both the diff and comments files to understand the scope, context, and any prior feedback. This helps you:
- Avoid duplicating issues already raised
- Assess whether prior feedback has been addressed
- Provide more targeted review insights

### Step 2: Launch Review Agents in Parallel

Run ALL of the following review tools **in parallel** as background bash commands. Each tool reviews the diff from a different perspective.

**Codex** (OpenAI) — focuses on correctness, bugs, security:

IMPORTANT: Embed the diff AND existing comments inline in the prompt. Codex cannot reliably read files — the `cat` must run in the host shell so content is passed directly as prompt text.

```bash
codex exec --dangerously-bypass-approvals-and-sandbox \
  "$(printf 'You are reviewing a PR diff. Review for correctness, bugs, security issues, and code quality. Focus on logic errors, edge cases, and performance issues. Consider the existing review comments below — do not duplicate issues already raised, and note if prior feedback has been addressed or not. Output your findings as a structured list with severity labels (critical/major/minor).\n\n--- DIFF ---\n'; cat pr-reviews/pr<PR_NUMBER>.diff; printf '\n\n--- EXISTING REVIEW COMMENTS ---\n'; cat pr-reviews/pr<PR_NUMBER>.comments 2>/dev/null || echo '(none)')"
```

**Gemini** (Google) — focuses on architecture, best practices:

```bash
gemini --yolo -p \
  "Read the file pr-reviews/pr<PR_NUMBER>.diff (the PR diff) and pr-reviews/pr<PR_NUMBER>.comments (existing review comments, if any). Review the diff for correctness, architecture patterns, and best practices. Consider the existing comments — do not duplicate issues already raised, and note if prior feedback has been addressed. Output your findings as a structured list with severity labels (critical/major/minor)."
```

**OpenCode (GLM-5)** — focuses on correctness, performance:

```bash
opencode run -m bailian-coding-plan/glm-5 \
  "Read the file pr-reviews/pr<PR_NUMBER>.diff (the PR diff) and pr-reviews/pr<PR_NUMBER>.comments (existing review comments, if any). Review the diff for correctness, performance, and maintainability. Consider the existing comments — do not duplicate issues already raised. Output your findings as a structured list with severity labels (critical/major/minor)."
```

**OpenCode (Kimi-K2.5)** — alternative perspective:

```bash
opencode run -m bailian-coding-plan/kimi-k2.5 \
  "Read the file pr-reviews/pr<PR_NUMBER>.diff (the PR diff) and pr-reviews/pr<PR_NUMBER>.comments (existing review comments, if any). Review the diff for correctness, performance, and maintainability. Consider the existing comments — do not duplicate issues already raised. Output your findings as a structured list with severity labels (critical/major/minor)."
```

### Step 3: Wait for ALL Agents, Then Synthesize

**IMPORTANT**: Wait for ALL launched review tools to complete before posting the review. Do NOT post with partial results. If a tool takes more than 5 minutes with no output progress, note it as "did not complete" and proceed with the rest.

After all tools finish:

1. **Collect** findings from each tool
2. **Cross-reference with existing comments**: skip issues already raised in prior reviews; note if prior feedback has been addressed by the current diff
3. **Categorize** new issues by severity:
   - **Critical**: bugs, security issues, data corruption risks
   - **Major**: logic errors, performance problems, API misuse
   - **Minor**: style, naming, documentation
4. **Identify consensus**: issues flagged by 2+ tools are high-confidence
5. **Dismiss false positives**: if a finding is clearly wrong given project context, dismiss it with reasoning
6. **Add your own review insights** based on your reading of the diff, existing comments, and project knowledge

### Step 4: Post Review

Post a single synthesized review comment on the PR via `gh pr review`:

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
| Finding | Codex | Gemini | GLM-5 | Kimi-K2.5 |
|---------|-------|--------|-------|-----------|
| [issue] | x     |        | x     |           |

---
Reviewed by: Claude Code + Codex + Gemini + GLM-5 + Kimi-K2.5
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
