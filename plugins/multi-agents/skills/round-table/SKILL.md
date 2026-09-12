---
name: round-table
description: Moderate a requested multi-AI panel discussion with distinct roles, follow-up rounds, and a saved transcript.
user-invocable: true
version: 0.8.0
---

# Round-Table Discussion

Resolve participants and `--skip <name>` with
[agent-resolution](../../references/agent-resolution.md). Choose complementary roles suited
to the topic, including a useful dissenting perspective. Show the roster and begin the
requested discussion; ask about roles only when that choice materially needs user input.

Use [shared conventions](../../references/agent-catalog.md) and only the selected CLI
references. External panelists discuss and review; they do not modify the user's project.

## Rounds and completion

- Ask each panelist a focused question in parallel. Give its role, the topic, and enough
  context to answer independently; do not ask it to simulate other panelists.
- Keep separate session IDs for each participant. Resume that exact session in later rounds,
  sending the prior synthesis, new question, and user steering. If resume fails, start a fresh
  session with a concise context summary.
- Present attributed responses, contribute your own perspective, then synthesize agreements,
  disagreements, and evidence without favoring your contribution.
- Follow the user's requested round count, pacing, or budget. If none is specified, complete
  one round and offer continuation or steering. End at the requested limit or when the user
  stops; do not insert approval pauses between already requested rounds.

Save a transcript in `round-table-<topic>-<timestamp>.md` (or the user's chosen path), with
participants/roles, each round's question and attributed answers, synthesis, and final
conclusions/open questions. Record participant failures and avoid presenting partial
participation as consensus. Writing the transcript does not authorize committing or pushing it.
