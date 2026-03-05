# Agent Resolution

This procedure builds the agent roster used by all skills. Run it before any other steps.

## Steps

1. **Read configuration**: Read CLAUDE.md files (project-level first, then user-level) and look for a `## Multi-Agents` section. Parse each `- {cli-name}` or `- {cli-name}: {model}` line into a roster entry. Project-level takes precedence — if found, skip user-level.

2. **Auto-detect fallback**: If no `## Multi-Agents` section is found in any CLAUDE.md, auto-detect available CLIs:
   ```bash
   which codex gemini opencode pi qwen 2>&1
   ```
   Add one default entry (no model) for each CLI found on `$PATH`.

3. **Load agent details**: Read `references/agent-catalog.md` for shared conventions and display name rules. Then, for each unique CLI in the roster, read `references/{cli-name}.md` to load that agent's command templates, prompt passing strategy, session resume details, output cleanup rules, and known quirks.

4. **Build display names**: Apply the catalog's display name rules to each roster entry.

5. **Apply exclusions**: If the user's arguments contain `--skip {name}` (where `{name}` is a CLI name, model name, or display name — case-insensitive), remove matched agents from the roster. Multiple `--skip` flags can be used together.

6. **Verify CLIs**: Run `which {all_cli_binaries_in_roster} 2>&1` to verify each CLI is installed. Remove any entries whose CLI is not found and note the missing CLIs.
