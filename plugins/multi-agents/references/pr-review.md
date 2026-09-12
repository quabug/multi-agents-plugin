# GitHub Review Operations

Use `gh` or an available GitHub API tool for the verified repository/PR. Fetch current PR
metadata and discussion before acting. Paginate comment/thread endpoints when needed.
`gh pr-review` is optional; inspect its installed help rather than assuming flags or schemas.

For read-only context, `gh pr view` and `gh pr diff` supply metadata and the remote diff.
For unpushed repairs, compare the fetched base with local HEAD and account for intended working
tree changes. Keep the exact reviewed head SHA with findings.

## Authorized publication

Posting a review or reply requires user authorization; a review-only request returns findings
in the conversation. Prepare the full review first. Use structured API arguments or a body
file (for example `gh pr review <pr> --comment --body-file <path>`) to preserve text literally.

For inline comments, verify path, side, and an actual diff line on the current head; proximity
to a changed line is not enough. Place findings that cannot map to the diff in the review body.
Use a pending review when supported, then submit once. If adding comments fails, inspect and
remove only a draft created by this task before falling back, avoiding duplicate submissions.

Read back the submitted review/reply to verify persistence. Resolve only fully addressed
threads when requested, after the fixing commit reaches the PR. Report operation failures;
continue independent repair work without claiming that unresolved threads were resolved.
