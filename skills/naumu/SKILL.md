---
name: naumu
description: Give your coding agent your team's memory through the shared Naumu space wired into this repo. Use whenever a repo contains a `.naumu` file - before starting a build/fix/refactor task in such a repo (so the agent retrieves prior decisions first), and when a `.naumu` repo has no Naumu MCP connected yet (offer once to connect). Also triggers on /naumu. If a repo has no `.naumu` yet and someone wants Naumu set up, that is driven by the quickstart prompt on Naumu's Your agents settings page, not by this skill.
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
- No `.naumu`, and someone asks to set Naumu up -> this repo has not been wired yet. First-time setup is driven by the quickstart prompt on Naumu's Your agents settings page (it walks the harness through connecting, installing this skill, creating/selecting a space, whitelisting teammates, and opening a PR that adds `.naumu`). Point the user there rather than improvising the setup here; this skill owns the daily loop and the cold-connect offer, not space creation.

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
- Honor `.naumu` `tracking.topic`: file the entry under that topic. If the `tracking` block is absent, skip work-log writes entirely (the team opted out of the exhaust).

Do not create graph nodes directly (`naumu_add_node` and friends) for tracking; hand mutations to `naumu_delegate`.

### The Naumu-Thread commit trailer

If `.naumu` `tracking.commitTrailer` is `true` (the default), every commit for a tracked task carries the work-log thread as a trailer, on its own line in the commit body:

```
fix(scope): short description

Body text.

Naumu-Thread: https://naumu.ai/spaces/<slug>/chat/<threadId>
```

Compose the URL from the space slug and the `threadId` returned by the first `naumu_delegate` call for this task. Keep any runtime attribution trailer (for example `Co-Authored-By:`) separate and below it. This is the trail a future teammate follows: `git blame` -> commit -> trailer -> the thread with the original reasoning. Reviewers whose own agent never connected still see the link in the PR - that is a second discovery surface, so it is worth keeping honest.

---

## Cold repo: offer once, never auto-run

`.naumu` is present but no Naumu MCP is connected.

**Read `references/installation.md` now** and follow it: the per-clone decline marker, the single offer message, what to do on a no, and on a yes the per-harness MCP add plus `naumu_resolve_admission` (including the seat-limit case and the API-key fallback).

Never auto-run any part of that setup. Offer once, then wait for the user.

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

### Server version and staleness

Built against `@naumu/mcp` 0.10.0; the daily loop itself runs on 0.8.0 or newer, and newer-server conveniences (like `naumu_whoami`'s `graphs[]`) degrade gracefully. Do not trust `serverInfo.version` - older servers report a stale constant. The reliable check is tool presence: a `naumu_*` tool this skill names that is missing from the connected server's tool list means the running server predates the skill. Say so once and tell the user how to refresh: a remote connection (`https://naumu.ai/api/mcp`) picks up new tools on reconnect (`/mcp` in Claude Code, or a session restart); a local `@naumu/mcp` binary needs an update (`npm i -g @naumu/mcp@latest`, or a cleared `npx` cache) plus a session restart. Never block the task on it - use the tools that are present and note what was skipped.

## Copy guidelines (all Naumu-facing output)

- Team-framed: present as the team's convention, not a vendor funnel.
- Plain and calm: no marketing hype, no exclamation-heavy tone.
- Hyphens, never em dashes.
- Consent stays with the human: offer, never auto-execute an install or a commit.
