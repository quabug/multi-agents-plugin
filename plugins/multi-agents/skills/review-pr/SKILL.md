---
name: review-pr
description: Review a GitHub pull request with multiple external AI reviewers and synthesize evidence-backed findings.
version: 0.8.0
---

# Multi-Agent PR Review

Identify the repository and PR from the supplied URL or number. Capture its base/head SHA,
diff, and existing review discussion. A review-only request does not require changing the
checkout. Preserve related comments and filter status/coverage noise.

Resolve the requested reviewers and `--skip <name>` via
[agent-resolution](../../references/agent-resolution.md). Follow
[shared conventions](../../references/agent-catalog.md), loading only their CLI references.
Split large reviews by coherent subsystem when helpful, recording any unreviewed scope.

Give reviewers the relevant diff, code, and prior feedback. Request concrete regressions,
severity, file/line evidence, and remaining uncertainties. Independently verify actionable
findings; agreement between agents is useful corroboration, not proof. Deduplicate findings
and report actual defects before optional suggestions.

Finish after all selected reviewers complete or reach the task's timeout. Disclose missing
coverage and validation gaps; failed/empty output is not a clean review. Return the review
in the conversation. Post it to GitHub only when the user has authorized posting; read
[PR review operations](../../references/pr-review.md) at that point. Code fixes, commits,
pushes, and merges require corresponding task scope.
