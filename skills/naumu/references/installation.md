# Cold repo: offer once, never auto-run

Read this when a repo has a `.naumu` file but no Naumu MCP server is connected. Offer to connect - once per clone - and never run anything without a yes.

## 1. Check the decline marker first

Before offering, check whether this clone already declined. The marker is keyed to the repo's origin URL so it is per repo and never committed:

```bash
# key = sha256 of the origin remote URL (fallback: the repo's absolute path)
key=$( { git remote get-url origin 2>/dev/null || git rev-parse --show-toplevel; } | tr -d '\n' | shasum -a 256 | cut -d' ' -f1 )
test -f "$HOME/.config/naumu/declined/$key" && echo declined
```

If the marker exists, stay silent about Naumu for this repo. A different repo is a fresh decision.

## 2. Make the offer (one short, team-framed message)

If not declined and not connected, say once, plainly:

> This repo logs work to the team's Naumu space (see `.naumu`). Want me to set up the connection so I can read the team's context and record my work? I will not run anything without your go-ahead.

Then stop and wait. Do not pitch again this session, and never re-offer in a repo you already offered in unless the user brings it up.

## 3. On no: write the marker, drop it

```bash
mkdir -p "$HOME/.config/naumu/declined"
touch "$HOME/.config/naumu/declined/$key"
```

Then never mention Naumu in this repo again.

## 4. On yes: connect, then resolve admission

Run the MCP-add command for the current harness (see below), which opens a browser for OAuth. The login page doubles as signup for someone without an account, so no account is required up front and no secret is copied.

Once connected, call `naumu_resolve_admission` with the space id from `.naumu`, passing the developer's `git config user.email` as `gitEmailHint`. This runs the team's admission tiers and either admits the developer to the space or files a request-to-join a member can approve. Report the outcome plainly (joined, or request sent and pending a member's approval).

If the response is `request-created` (or `request-pending`) with `reason: seat-limit`, the developer matched a whitelist or domain rule but the space is at its seat limit. Tell them plainly: they would have joined automatically, but the space is full, so their request is now waiting for an admin to approve them or upgrade the plan. Do not report this as "no match".

`gitEmailHint` is a hint for the approver, never authentication - it just lets a join-request card show "found in this repo's git history".

## Per-harness MCP add

Tell the agent's own harness to add the remote server; the agent knows which harness it is running in.

- Claude Code: `claude mcp add --transport http --scope user naumu https://naumu.ai/api/mcp`. Keep `--scope user`; without it the server is registered only for the current directory. Config is read at startup, so a new session is needed before `/mcp` lists it.
- Cursor: add a remote MCP entry pointing at `https://naumu.ai/api/mcp` in the MCP settings - use the global `~/.cursor/mcp.json` so it applies to every project, rather than a repo-local `.cursor/mcp.json`; OAuth completes in the browser.
- Codex: `codex mcp add naumu --transport http https://naumu.ai/api/mcp`. If the harness cannot complete browser OAuth, fall back to the API-key + stdio path (see Fallback).
- Other harnesses: use the equivalent "add remote/HTTP MCP" command with the same URL.

**Fallback (API key + stdio)** for harnesses without remote-OAuth: create a key in Naumu account settings and configure the stdio server:
`npx -y -p @naumu/mcp naumu-mcp` with `NAUMU_API_KEY=nmu_...` in its env. Reserve this for CI, bots, and harnesses that cannot do remote OAuth.
