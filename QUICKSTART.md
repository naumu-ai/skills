# Quickstart

Paste the prompt below into your coding agent (Claude Code, Codex, Cursor, or any harness with MCP support). It drives the whole first-time setup: connecting the Naumu MCP server, installing the daily `naumu` skill, creating or picking a team space, whitelisting teammates from git history, and opening a PR that wires the repository in - asking before every command. The same text is offered as a copy block on the Linked apps settings page in the app (via `NAUMU_QUICKSTART_PROMPT`); keep every copy byte-identical.

Set up Naumu for this repository. Ask me before running each command, and never commit directly to a protected branch.
1. Connect: if Naumu MCP tools are not available, add the server for this harness, then complete sign-in in the browser (creating my account if needed): Claude Code: claude mcp add --transport http naumu https://naumu.ai/api/mcp - Codex: codex mcp add naumu --transport http https://naumu.ai/api/mcp - Cursor: add "naumu": {"url": "https://naumu.ai/api/mcp"} under mcpServers in .cursor/mcp.json
2. Install the Naumu skill for day-to-day use: npx skills add naumu-ai/skills
3. Call naumu_whoami. If I already belong to a team space, ask which space this repo should use; otherwise create one with naumu_create_graph.
4. Scan git history for teammates: git log --all --format='%ae|%an|%at', keep authors active in the last 90 days, drop bots and noreply addresses, rank by commit count then recency. Show me the list - addresses on public providers (gmail and similar) go in a separate group I must confirm individually.
5. After I confirm, call naumu_whitelist_members with the confirmed emails and repoInit true. Whitelisted teammates auto-join the space when they sign in with that email - no invite emails are sent.
6. Create .naumu in the repo root: { "space": "<space id>", "track": { "workLog": true, "commitTrailer": true } }
7. Append to AGENTS.md: This repo logs work to the team's Naumu space (see .naumu). If Naumu MCP tools are missing, offer the user setup once: npx skills add naumu-ai/skills, then connect and call naumu_resolve_admission with the space id. Offer only - never auto-run.
8. Open a pull request with these two changes so the team can review.
9. Offer to seed the space with a short architecture and glossary summary of this repo so first searches return something useful.
