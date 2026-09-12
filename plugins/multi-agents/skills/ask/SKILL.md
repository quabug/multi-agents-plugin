---
name: ask
description: Ask selected external AI agents a question and compare their answers. Use for a requested multi-AI consultation or a named agent's opinion.
user-invocable: true
version: 0.8.0
---

# Multi-Agent Ask

Resolve the requested agents with [agent-resolution](../../references/agent-resolution.md).
Accept `/ask <question>`, `/ask <agent-or-model> <question>`, and `--skip <name>`.
A leading CLI name, model suffix, `cli:model`, or display name filters the roster when it
matches; otherwise retain it as part of the question. Honor an explicitly named target;
report an unavailable target instead of silently asking unrelated agents.

Send the user's question to selected agents in parallel using the applicable CLI reference.
Preserve its wording; distinguish routing arguments from question text. Follow
[shared conventions](../../references/agent-catalog.md) for permissions and result collection.

Attribute each useful answer, identify failed participants, and compare substantive agreements
and disagreements. Add your own assessment when useful. An empty or failed response is not an
opinion; if every participant fails, disclose that and give your own answer.
