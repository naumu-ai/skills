---
name: naumu
description: Give your coding agent your team's memory through the shared Naumu space wired into this repo. Use whenever a repo contains a `.naumu` file - before starting a build/fix/refactor task in such a repo (so the agent retrieves prior decisions first), and when a `.naumu` repo has no Naumu MCP connected yet (offer once to connect). Also triggers on /naumu. If a repo has no `.naumu` yet and someone wants Naumu set up, that is driven by the quickstart prompt on Naumu's Linked apps settings page, not by this skill.
---

# Naumu

Naumu is a shared knowledge graph ("space") a team wires into a repo so every developer's agent can read the team's memory (prior decisions, module rationale, who changed what and why) and write a light audit trail of its own work back into it.

A repo opts in by committing a `.naumu` file (the space id, nothing secret) and a short AGENTS.md pointer. This skill is the behavior behind that pointer: the daily read+write loop, plus the one-time offer to connect a cold clone.

## When this fires

- A session opens in a repo that has a `.naumu` file at its root.
- Before you start a substantive task (build, fix, refactor, add a feature) in a `.naumu` repo, so you can retrieve relevant context first.
- A `.naumu` repo is present but no Naumu MCP server is connected (offer once, below).

Decide which mode you are in:

- `.naumu` present AND a Naumu MCP server is connected -> Connected repo (read+write loop below).
- `.naumu` present AND no Naumu MCP -> Cold repo (offer once, never auto-run).
- No `.naumu`, and someone asks to set Naumu up -> this repo has not been wired yet. First-time setup is driven by the quickstart prompt on Naumu's Linked apps settings page (it walks the harness through connecting, installing this skill, creating/selecting a space, whitelisting teammates, and opening a PR that adds `.naumu`). Point the user there rather than improvising the setup here; this skill owns the daily loop and the cold-connect offer, not space creation.

Everything here is team tooling. Copy is team-framed and plain: no hype, no exclamation runs, hyphens not em dashes.

---

## Connected repo: the read+write loop

The deal for the individual developer is retrieval: their agent starts each task already knowing what the team knows. The exhaust (work-log threads, commit trailers) is the team's audit trail. Read first, work, then write a short record.

### Read: pre-task retrieval reflex (hard latency budget)

Before starting a non-trivial task, run ONE fast scoped lookup against the space to surface prior decisions and related work. This is a reflex, not a research project.

- Read the space id from `.naumu` (`space` field).
- Use `naumu_search` with a query built from the task (feature name, file/module, symbols). It returns nodes by meaning, fast.
- If a result points at a conversation worth the context, follow up with `naumu_read_thread` on that thread only.
- Fold anything relevant (a prior decision, a gotcha, a linked ticket) into how you approach the task, and mention it to the user in one line.

Budget: this pre-task check must not block the task for more than about 5 seconds. `naumu_search` and a single `naumu_read_thread` fit that. If a call is slow or errors, drop it and start the work - never stall the task on retrieval.

Do NOT use `naumu_ask` for the reflex. `naumu_ask` runs the full @Naumu agent and takes tens of seconds; it is for explicit user questions ("ask the team space whether we already solved X"), not the every-task preamble. Reserve it for when the user actually asks a question of the space.

### Write: work-log on completion

When a unit of work is done (or hits a milestone worth recording), post a short work-log entry into the space so reviewers and future agents can find it.

- Preferred: `naumu_delegate` with the space id and a bulleted task describing what was done. This records the work and lets @Naumu file/relate it. On a first entry it returns a `threadId`; reuse that same `threadId` for later updates on the same task rather than opening new threads.
- Alternative when you only need to append a plain note to an existing thread without summoning the agent: `naumu_post_message`.
- Keep entries bulleted and short (one bullet per concrete change), never a wall of prose - a human scrolls these.
- Honor `.naumu` `track.workLog`: if it is `false`, skip work-log writes entirely (the team opted out of the exhaust).

Do not create graph nodes directly (`naumu_add_node` and friends) for tracking; hand mutations to `naumu_delegate`.

### The Naumu-Thread commit trailer

If `.naumu` `track.commitTrailer` is `true` (the default), every commit for a tracked task carries the work-log thread as a trailer, on its own line in the commit body:

```
fix(scope): short description

Body text.

Naumu-Thread: https://naumu.ai/spaces/<slug>/chat/<threadId>
```

Compose the URL from the space slug and the `threadId` returned by the first `naumu_delegate` call for this task. Keep any runtime attribution trailer (for example `Co-Authored-By:`) separate and below it. This is the trail a future teammate follows: `git blame` -> commit -> trailer -> the thread with the original reasoning. Reviewers whose own agent never connected still see the link in the PR - that is a second discovery surface, so it is worth keeping honest.

---

## Cold repo: offer once, never auto-run

`.naumu` is present but no Naumu MCP is connected. Offer to connect - once per clone - and never run anything without a yes.

### 1. Check the decline marker first

Before offering, check whether this clone already declined. The marker is keyed to the repo's origin URL so it is per repo and never committed:

```bash
# key = sha256 of the origin remote URL (fallback: the repo's absolute path)
key=$( { git remote get-url origin 2>/dev/null || git rev-parse --show-toplevel; } | tr -d '\n' | shasum -a 256 | cut -d' ' -f1 )
test -f "$HOME/.config/naumu/declined/$key" && echo declined
```

If the marker exists, stay silent about Naumu for this repo. A different repo is a fresh decision.

### 2. Make the offer (one short, team-framed message)

If not declined and not connected, say once, plainly:

> This repo logs work to the team's Naumu space (see `.naumu`). Want me to set up the connection so I can read the team's context and record my work? I will not run anything without your go-ahead.

Then stop and wait. Do not pitch again this session, and never re-offer in a repo you already offered in unless the user brings it up.

### 3. On no: write the marker, drop it

```bash
mkdir -p "$HOME/.config/naumu/declined"
touch "$HOME/.config/naumu/declined/$key"
```

Then never mention Naumu in this repo again.

### 4. On yes: connect, then resolve admission

Run the MCP-add command for the current harness (see below), which opens a browser for OAuth. The login page doubles as signup for someone without an account, so no account is required up front and no secret is copied.

Once connected, call `naumu_resolve_admission` with the space id from `.naumu`, passing the developer's `git config user.email` as `gitEmailHint`. This runs the team's admission tiers and either admits the developer to the space or files a request-to-join a member can approve. Report the outcome plainly (joined, or request sent and pending a member's approval).

If the response is `request-created` (or `request-pending`) with `reason: seat-limit`, the developer matched a whitelist or domain rule but the space is at its seat limit. Tell them plainly: they would have joined automatically, but the space is full, so their request is now waiting for an admin to approve them or upgrade the plan. Do not report this as "no match".

`gitEmailHint` is a hint for the approver, never authentication - it just lets a join-request card show "found in this repo's git history".

### Per-harness MCP add

Tell the agent's own harness to add the remote server; the agent knows which harness it is running in.

- Claude Code: `claude mcp add --transport http naumu https://naumu.ai/api/mcp`
- Cursor: add a remote MCP entry pointing at `https://naumu.ai/api/mcp` in the MCP settings (`.cursor/mcp.json`); OAuth completes in the browser.
- Codex: `codex mcp add naumu --transport http https://naumu.ai/api/mcp`. If the harness cannot complete browser OAuth, fall back to the API-key + stdio path (see Fallback).
- Other harnesses: use the equivalent "add remote/HTTP MCP" command with the same URL.

**Fallback (API key + stdio)** for harnesses without remote-OAuth: create a key in Naumu account settings and configure the stdio server:
`npx -y -p @naumu/mcp naumu-mcp` with `NAUMU_API_KEY=nmu_...` in its env. Reserve this for CI, bots, and harnesses that cannot do remote OAuth.

---

## Tool reference

Use only tools the connected server exposes; resolve by capability if a prefix differs by harness.

| Need | Tool |
|------|------|
| Identity / spaces | `naumu_whoami`, `naumu_list_graphs`, `naumu_list_topics` |
| Fast pre-task retrieval | `naumu_search`, then `naumu_read_thread` on a specific thread |
| Explicit question of the space (slow, on request only) | `naumu_ask` |
| Record work / hand over a mutation | `naumu_delegate` (append plain note: `naumu_post_message`) |
| Connect a cold clone (membership) | `naumu_resolve_admission`, `naumu_admission_status` |

- `naumu_admission_status(graphId)` - for a member: current whitelist, domain wildcards, and pending join-request count/list. Use to check who is waiting.
- `naumu_whoami` - identity, plus on newer servers a `graphs[]` list (each entry carries a `kind` of `personal` or `team` and an `isPersonalDefault` flag). When the list is present, use it to pick which space to use (a `team` space, not the personal default) without a second call; if a server omits the list, fall back to `naumu_list_graphs`.

## Copy guidelines (all Naumu-facing output)

- Team-framed: present as the team's convention, not a vendor funnel.
- Plain and calm: no marketing hype, no exclamation-heavy tone.
- Hyphens, never em dashes.
- Consent stays with the human: offer, never auto-execute an install or a commit.
