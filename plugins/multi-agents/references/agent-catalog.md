# Shared Agent Conventions

This plugin consults external CLI agents. Use the user's selected roster/models and current
CLI help when a reference's compatibility example differs from the installed version.

## Scope and execution

- Question, discussion, and review participants are read-only. Use supported tool restrictions
  or an isolated workspace containing only needed artifacts. Do not enable unrestricted or
  auto-approval modes merely because an old example used them; stay within current task
  permissions. A PR-repair host owns authorized edits and publication.
- Dispatch independent participants concurrently using the host's available process tools.
  Use unique output/session locations per participant. Bound waiting by the user's budget;
  absent one, ten minutes per external call is a reasonable default. Poll in short intervals
  so steering and status updates remain possible; cancel only processes started for this task.
- Pass prompts as data through stdin, files, or properly quoted arguments. Never interpolate
  question/diff text into executable shell code or a `printf` format string.
- Capture exit status and stderr. Strip display metadata/ANSI noise while preserving errors
  that affect confidence. Use explicit session IDs, never a shared latest-session shortcut.

## Results

Label the host contribution as "Claude Code", distinct from any external Codex participant.

Collect each result independently. If the host cancels sibling polls when one fails, poll
sequentially (Claude Code `TaskOutput`) or inspect separate output files. A failed, truncated, or empty response is
missing coverage; it is not evidence that no issues exist. Skip persistently failing optional
participants and disclose the gap rather than restarting the entire consultation.

For reviews, retain actionable human/bot findings and filter status reports and quota noise.
Verify findings against code; consensus alone does not establish correctness. Report severity,
location, consequence, and relevant evidence. Do not duplicate already addressed feedback.

Use [pr-review.md](pr-review.md) only for retrieving structured review threads or authorized
GitHub review/comment mutations. The plugin does not require installing a review extension.
