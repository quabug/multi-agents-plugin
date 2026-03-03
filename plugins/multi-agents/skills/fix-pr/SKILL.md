---
name: fix-pr
description: This skill should be used when the user asks to "fix a PR", "fix pull request", "fix PR issues", "review and fix PR", or wants an iterative multi-agent review + fix loop that automatically resolves issues found in a GitHub pull request.
version: 0.6.0
---

# Multi-Agent PR Fix

Review a GitHub PR using multiple AI agents in parallel, then iteratively fix the issues found until the PR is clean. Unlike `review-pr` which only posts a review comment, this skill actively modifies code to resolve issues.

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

Fetch the PR metadata and check out the PR branch first:

```bash
gh pr view <PR_NUMBER> --json title,body,baseRefName,headRefName
gh pr checkout <PR_NUMBER>
```

After checkout, detect the `gh pr-review` extension and store flags:

```bash
REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner)
gh pr-review --version 2>/dev/null
```

Store `has_pr_review_ext` (true if exit code 0, false otherwise) and `REPO`. Refer to the **PR Review Extension** section in `references/agent-catalog.md` for the full command reference.

Check for uncommitted local changes:

```bash
git status --short
```

If there are uncommitted changes that overlap with the PR files, warn the user — these may represent prior fixes that haven't been committed yet. Ask whether to include them or stash them.

Generate the diff locally against the base branch (more reliable than `gh pr diff` which only reflects the remote state):

```bash
mkdir -p pr-reviews
BASE_BRANCH=$(gh pr view <PR_NUMBER> --json baseRefName -q .baseRefName)
git diff ${BASE_BRANCH}...HEAD > pr-reviews/pr<PR_NUMBER>.diff
```

Fetch existing review comments and conversations — these are first-class issues to fix. Use one of two paths depending on `has_pr_review_ext`:

**When `has_pr_review_ext` is true** — use structured JSON for review threads plus REST for issue-level comments:

```bash
gh pr-review review view -R ${REPO} --pr <PR_NUMBER> --unresolved > pr-reviews/pr<PR_NUMBER>.review-threads.json
gh api repos/${REPO}/issues/<PR_NUMBER>/comments --jq '.[] | "[\(.user.login)] — \(.body)"' > pr-reviews/pr<PR_NUMBER>.comments 2>/dev/null
```

The `review-threads.json` contains structured data with `thread_id`, `path`, `line`, `body`, and `is_resolved` for each thread. This is preferred over REST for better parsing and thread ID access.

**Fallback (when `has_pr_review_ext` is false)** — current REST API approach:

```bash
gh api repos/${REPO}/pulls/<PR_NUMBER>/comments --jq '.[] | "[\(.user.login)] \(.path):\(.line // .original_line) — \(.body)"' > pr-reviews/pr<PR_NUMBER>.comments 2>/dev/null
gh api repos/${REPO}/pulls/<PR_NUMBER>/reviews --jq '.[] | select(.body != "") | "[\(.user.login)] (\(.state)) — \(.body)"' >> pr-reviews/pr<PR_NUMBER>.comments 2>/dev/null
gh api repos/${REPO}/issues/<PR_NUMBER>/comments --jq '.[] | "[\(.user.login)] — \(.body)"' >> pr-reviews/pr<PR_NUMBER>.comments 2>/dev/null
```

Read the diff and comments/threads files to understand the scope, context, and any prior feedback.

### Step 2: Gather Existing Feedback

Parse the fetched comments into an initial issues list. Apply the **PR Comment Filtering** rules from the catalog to skip CI bot noise, quota messages, and auto-generated summaries. Focus on actionable feedback from human reviewers and code review bots with severity-tagged findings.

**When `has_pr_review_ext` is true**: Parse `review-threads.json` to extract structured issues. For each unresolved thread, preserve the `thread_id` alongside the issue data. Maintain a mapping for each actionable issue:

```
{thread_id, severity, path, line, description}
```

This mapping is persisted across loop iterations so that threads can be resolved after fixes are applied (Step 3g).

**Fallback**: Parse the plain-text `.comments` file as before (no thread IDs available).

Categorize each piece of actionable feedback by severity:
- **Critical**: bugs, security issues, data corruption risks mentioned by reviewers
- **Major**: logic errors, performance problems, API misuse, requested changes
- **Minor**: style, naming, documentation suggestions

These existing review comments are treated as first-class issues alongside multi-agent findings — the skill should attempt to address unresolved feedback from human reviewers too.

### Step 3: Review-Fix Loop (max 3 iterations)

Track iteration count starting at 1. For each iteration:

#### 3a. Launch Multi-Agent Review

Run ALL agents from the resolved roster **in parallel** using `run_in_background: true` on each Bash call. Each agent reviews the current diff.

For each agent in the roster, build the review command using the agent's **one-shot** command template from the catalog:

- **Agents that cannot reliably read files** (e.g., Codex per the catalog): Embed the diff AND existing comments inline in the prompt via `printf` + `cat`.
- **Agents that can reference files** (e.g., Gemini, OpenCode per the catalog): Reference the diff and comments files by path in the prompt.
- **Agents with a model override**: Include the model flag (e.g., `-m {model}` for OpenCode).
- **Set `timeout: 120000`** (120 seconds) on each Bash call per the catalog's common conventions.

**Review prompt** (adapt per agent's prompt passing strategy from the catalog):

```
Review the PR diff for correctness, bugs, security issues, architecture patterns, performance, and code quality.
Focus on logic errors, edge cases, and best practices.
Consider the existing review comments below — do not duplicate issues already raised, and note if prior feedback has been addressed or not.
Output your findings as a structured list with severity labels (critical/major/minor).
If the code looks clean with no significant issues, explicitly state "No critical or major issues found."

--- DIFF ---
{diff content or file reference}

--- EXISTING REVIEW COMMENTS ---
{comments content or file reference}
```

#### 3b. Collect & Synthesize Findings

Collect results following the **Result Collection** and **Output Validation** rules from the catalog: collect `TaskOutput` sequentially (never in parallel), handle timeouts with `TaskStop`, and discard useless output without treating it as "no issues found."

After all tools finish (or time out):

1. **Collect** findings from each tool. Apply output cleanup rules from the catalog.
2. **Cross-reference with existing comments**: skip issues already raised in prior reviews; note if prior feedback has been addressed by the current diff.
3. **Merge with existing PR feedback**: deduplicate against issues already raised by human reviewers, and include unresolved human feedback as additional issues to fix.
4. **Categorize** all issues by severity:
   - **Critical**: bugs, security issues, data corruption risks
   - **Major**: logic errors, performance problems, API misuse
   - **Minor**: style, naming, documentation
5. **Identify consensus**: issues flagged by 2+ tools are high-confidence
6. **Dismiss false positives**: if a finding is clearly wrong given project context, dismiss it with reasoning
7. **Add your own independent review**: Review the diff yourself as an additional reviewer. Identify issues the agents may have missed, validate or challenge their findings, and contribute your own perspective on correctness, architecture, and edge cases. Your findings are merged into the issues list on equal footing with agent findings

#### 3c. Check Exit Condition

If there are **no critical or major issues** (from agents or existing reviews), exit the loop and proceed to Step 4 (Final Report).

#### 3d. Present Issues & Ask User

Display the categorized issues to the user. Use `AskUserQuestion` to confirm before applying fixes:

- **Continue** — fix all critical and major issues (Recommended)
- **Select issues** — let user specify which issues to fix
- **Stop** — exit the loop, report current state

If the user chooses "Stop", exit the loop and proceed to Step 4.

#### 3e. Fix Issues

Read the relevant source files and apply fixes for all critical and major issues (both from multi-agent review AND existing PR feedback). Only fix critical and major issues — minor issues are listed but not auto-fixed to avoid churn.

For each fix:
1. Read the affected file(s) to understand the full context
2. Apply the fix using appropriate edit tools
3. Verify the fix doesn't introduce new issues

**Run tests before committing** if the project has a test command (check CLAUDE.md or common patterns like `dotnet test`, `npm test`, `pytest`, `go test`, etc.). If tests fail, fix the issues before committing.

After all fixes are applied and tests pass, create a commit. **Stage only the files you modified** — do NOT use `git add -A` or `git add .` which may accidentally include unrelated changes:

```bash
git add {list of specific files modified}
git commit -m "fix: address review findings (iteration {N})

- {brief description of each fix}

Fixes identified by multi-agent review ({display_name_1}, {display_name_2}, ...)"
```

#### 3f. Push & Update Diff

Push the fix commit to the remote so the PR is updated:

```bash
git push
```

Then re-generate the diff locally for the next review iteration:

```bash
git diff ${BASE_BRANCH}...HEAD > pr-reviews/pr<PR_NUMBER>.diff
```

When `has_pr_review_ext` is true, also refresh the thread data so the next iteration sees up-to-date resolution status:

```bash
gh pr-review review view -R ${REPO} --pr <PR_NUMBER> --unresolved > pr-reviews/pr<PR_NUMBER>.review-threads.json
```

#### 3g. Resolve Fixed Review Threads (when `has_pr_review_ext` is true)

**Only when `has_pr_review_ext` is true.** After pushing fixes, resolve the review threads for issues that were actually fixed in this iteration.

For each fixed issue that has a `thread_id`, run sequentially (NOT in parallel):

1. **Reply** to the thread with what was fixed:
   ```bash
   gh pr-review comments reply <PR_NUMBER> -R ${REPO} \
     --thread-id <THREAD_ID> --body "Fixed in $(git rev-parse --short HEAD). {brief description of the fix}"
   ```

2. **Resolve** the thread:
   ```bash
   gh pr-review threads resolve --thread-id <THREAD_ID> -R ${REPO} <PR_NUMBER>
   ```

**Rules:**
- Only resolve threads for issues actually fixed this iteration — skip minor issues that were not auto-fixed, partial fixes, and agent-only findings without thread IDs
- If a reply or resolve command fails, log the error and continue — never abort the fix loop over a resolution failure
- Thread resolution is best-effort and does not block subsequent iterations

Then loop back to step 3a for re-review.

### Step 4: Final Report

Display a summary to the user:

```
## Fix PR Summary

### Iterations: {count}

### Issues Found & Fixed
{for each iteration:}
**Iteration {N}:**
- {list of critical/major issues found and fixed}

### Remaining Minor Issues
{list of minor issues that were not auto-fixed, if any}

### Commits Made
{list of commit hashes and messages from each iteration}

### Status
{either "All critical/major issues resolved" or "Max iterations (3) reached — {N} issues remain"}
```

If max iterations (3) were reached and issues remain, ask the user if they want to continue with more iterations.

### Step 5: Cleanup

```bash
rm -f pr-reviews/pr<PR_NUMBER>.diff pr-reviews/pr<PR_NUMBER>.comments pr-reviews/pr<PR_NUMBER>.review-threads.json
rmdir pr-reviews 2>/dev/null
```

## Important Design Decisions

- **Max 3 iterations** by default to prevent infinite loops. Ask user if they want to continue after 3.
- **Only fix critical and major issues** — minor issues (style, naming) are reported but not auto-fixed to avoid churn.
- **Existing PR feedback is first-class** — unresolved comments from human reviewers are treated as issues to fix, merged and deduplicated with multi-agent findings.
- **Each iteration commits** — so fixes are incremental and reviewable.
- **No GitHub review comment posted** — unlike `review-pr`, this skill fixes instead of reporting. The fixes themselves are the output.
- **Thread resolution is best-effort** — `gh pr-review` thread resolution (Step 3g) never blocks the fix loop. If it fails, the fix was still applied.
- **Structured JSON preferred over REST** — when `gh pr-review` is available, structured thread data provides thread IDs for resolution and better parsing than the REST API's plain-text output.
- **User confirmation before fixing** — after each review round, show the issues found and ask the user to confirm before applying fixes.
- **Pushes after each commit** — keeps the remote PR up to date so reviewers can see incremental fixes.
- **Local diff generation** — always use `git diff <base>...HEAD` instead of `gh pr diff` to ensure the diff reflects local commits that haven't been pushed yet.
- **Run tests before committing** — verify fixes don't break anything before creating a commit.
- **Stage specific files** — never use `git add -A`; only stage the files that were actually modified for the fix.

## Error Handling

- If a tool is not installed, skip it and note in the findings
- If `gh` is not authenticated, prompt the user to run `gh auth login`
- If the PR number is invalid, report the error clearly
- If all external tools fail, provide your own review based on the diff and fix issues yourself
- If a fix introduces a build error, attempt to resolve it before committing
- For agent-specific error handling (model name errors, timeouts, useless output, result collection), refer to the catalog's Common Conventions section
- If `gh pr-review` extension commands fail during thread resolution (Step 3g), log the error and continue — thread resolution is best-effort and never blocks the fix loop
- If `gh pr-review review view` fails during comment fetching (Step 1), fall back to the REST API approach for that iteration
- If `gh pr-review` is installed but returns errors for a specific PR (e.g., permissions), silently fall back to REST-based comment fetching
