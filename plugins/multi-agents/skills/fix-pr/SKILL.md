---
name: fix-pr
description: Repair a GitHub pull request's review or CI findings, using external reviewers when a multi-agent repair is requested or useful.
version: 0.8.0
---

# PR Repair

Identify the PR, current base/head, failing checks, unresolved feedback, and checkout state.
Inspect local changes before switching branches. Use an isolated worktree when another task
owns the checkout; preserve existing work and avoid automatic stashing.

Resolve requested reviewers with [agent-resolution](../../references/agent-resolution.md).
Use [shared conventions](../../references/agent-catalog.md) for external review and load only
the selected CLI references. Existing actionable feedback is part of the repair scope.

## Repair and verify

Validate findings against current code and logs, then implement the requested repairs.
Address requested minor fixes too; avoid unrelated style churn. Give external reviewers
coherent diff/code sections and relevant prior feedback. Independently confirm their findings.

Run checks suited to the repaired behavior and the repository's real gates. Continue fixing
failures caused by the change. Repeat review where new changes or unresolved concerns justify
it; a third review round alone is not a reason to stop while a concrete repair is progressing.
Stop repeating an unchanged failed approach when there is no new evidence, and report the
specific dependency, permission, or environment blocker after useful alternatives are exhausted.

## Deliver the requested outcome

A request to update the PR includes committing and pushing the repair to its verified source
ref. Preserve any explicit local-only or no-push boundary. Stage only intended changes, refresh
the PR head/checks after pushing, and distinguish local proof from required CI success.

Reply to reviewers only when posting is authorized. Resolve only threads whose findings the
pushed repair actually addresses, when thread resolution is part of the task; use
[PR review operations](../../references/pr-review.md) for the API details. A failed thread
operation does not undo a code fix, but remains visible in the completion report.

If the user also requested merge, continue through required checks/review gates, recheck the
head and mergeability, and merge the verified head using repository conventions. Otherwise
finish at the requested local patch or updated PR. Report the repair, checks, PR/commit state,
and any remaining blocker; do not claim completion solely because a patch exists.
