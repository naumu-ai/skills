# skills-registry

Source for the public `naumu-ai/skills` registry. What lives here is exported to that GitHub repo and installed per developer with:

```bash
npx skills add naumu-ai/skills
```

That install ships three skills, `naumu`, `naumu-import` and `naumu-worklog`. The skill bodies are intentionally generic and public - no hardcoded team, space, or company. A repo opts in by committing two small artifacts (see `templates/`); the skills read them at runtime.

## The skills

- **`naumu`** (daily loop + cold-connect) - the behavior in a repo that has a `.naumu` file. It runs the pre-task retrieval reflex, writes the work-log + `Naumu-Thread` commit trailer, and - in a cold clone with no MCP connected - makes the one-time offer to connect (per-harness MCP add, `naumu_resolve_admission`, seat-limit relay, decline marker). It does not create spaces or write `.naumu`.
- **`naumu-import`** (one-off corpus import) - turns a folder or export (docs, ticket dumps with comment chains, images/video/PDF media) into structured content in the team's existing space. It surveys and triages before reading anything at scale, extends the space's schema rather than redesigning it, ingests in resumable chunks against a file-backed ledger, uploads and embeds media in the right notes and threads, and restructures only what it created. Built for corpora too large for one session; not a live sync - Naumu's built-in integrations cover continuous mirroring.
- **`naumu-worklog`** (work log outside a repo) - the Claude desktop / web / Cowork counterpart of the `naumu` work-log loop (ChatGPT uses the same skill + memory setup; apps without skills get the same loop as memory / custom instructions, built by `WORK_LOG_INSTRUCTIONS_TEMPLATE` in `apps/web/.../helpers/agent-work-log.ts` - change the two together), for non-coding work (marketing, research, planning). It opens a work-log thread at the start of a work session, follows up on decisions and posts a summary when done. Its target (space id + topic id - the `.naumu` equivalent) is a placeholder line in the skill. The Your agents settings page fills it in and wraps the result in one setup prompt: the user pastes it into Claude, which saves the skill and a memory line that makes it open the skill before any work worth logging (skills alone under-trigger - a research request never reads as "log my work"). The generic copy falls back to the same line in Claude's memory or project instructions. It stands down when a repo's `.naumu` / AGENTS.md rule or a Naumu thread already owns logging, so installing it next to `naumu` never double-logs. The web app serves the template at `apps/web/public/downloads/naumu-worklog.md`, byte-identical to `skills/naumu-worklog/SKILL.md` (`agent-work-log.test.ts` fails on drift); keep the placeholder line exactly as the test expects. Setup: https://naumu.ai/docs/agent-work-log.

Each skill states the minimum `@naumu/mcp` server version it needs in its SKILL.md. A referenced `naumu_*` tool missing from the connected server's tool list means the running server is stale - reconnect a remote server (`/mcp` in Claude Code, or restart the session) or update a local binary (`npm i -g @naumu/mcp@latest`), then reconnect.

First-time setup (create/select a space, whitelist teammates from git history, write `.naumu` + the AGENTS.md section, open a PR) is not a skill - it is driven by the pasteable quickstart prompt (`QUICKSTART.md`), which the app also surfaces as a copy block on the Your agents settings page. That keeps the on-disk skill purely about the day-to-day loop.

## Layout

```
skills-registry/
  README.md                          this file
  QUICKSTART.md                      the pasteable first-time-setup prompt (shipped in the app too)
  skills/naumu/SKILL.md              the daily skill (read+write loop, cold-connect etiquette)
  skills/naumu/references/           loaded on demand: installation.md (the cold-connect procedure)
  skills/naumu-import/               the one-off corpus import skill (slim SKILL.md + references/)
  skills/naumu-worklog/SKILL.md      the Claude desktop / web / Cowork work-log skill (served as naumu.ai/downloads/naumu-worklog.md)
  templates/naumu.json               the .naumu config template a repo commits
  templates/AGENTS-section.md        the ~4-line AGENTS.md pointer a repo commits
```

## How the loop closes

1. A developer installs the skill once (`npx skills add naumu-ai/skills`).
2. In a repo they own, they run the quickstart prompt (from the Your agents settings page). It creates or selects a space, whitelists teammates from git history, writes `.naumu` + the AGENTS.md section, and opens a PR. The PR also carries the installed skill artifacts - `.agents/skills/naumu` and the `.claude/skills/naumu` symlink the installer creates, which git tracks fine - so every teammate and every fresh clone gets skill discovery without re-running the installer.
3. The PR merges. Now every other teammate's harness reads the committed AGENTS.md pointer, which tells it to install the same skill; the daily `naumu` skill offers (once) to connect.
4. Connected teammates get the team's memory before each task and leave a work-log trail behind them.

The repo artifact installs the skill; the quickstart prompt installs the repo artifact. That is the whole wedge.

## Pinning, the lockfile, and local forks

- **Pin to a release tag** for a baseline that does not move under the team: `npx skills add naumu-ai/skills@vX.Y.Z`. A bare `npx skills add naumu-ai/skills` tracks `main` and shifts whenever we republish, which is fine for a solo developer and surprising for a repo full of them.
- **`computedHash` in `skills-lock.json` is a directory-manifest hash**, not a checksum of `SKILL.md`. The skills CLI feeds each file's relative path followed by its bytes into a single sha256, across every file in the skill folder sorted by path. Team tooling that compares `shasum -a 256 SKILL.md` against it reports a mismatch on every single install - verify with the CLI rather than reimplementing the hash.
- **To fork a skill on purpose**, drop its entry from `skills-lock.json`, or copy the skill out of the CLI-managed `.agents/skills/` and `.claude/skills/` paths first. `skills update` silently overwrites CLI-managed skill directories and offers no fork or eject affordance, so an un-ejected local edit is lost at the next update with no warning.

## The committed artifacts

### `.naumu` (from `templates/naumu.json`)

JSON cannot carry comments, so the fields are documented here:

```json
{
  "space": "<space-id>",
  "slug": "<space-slug>",
  "tracking": {
    "topic": "work-log",
    "topicId": "<topic-id>",
    "commitTrailer": true
  }
}
```

- `space` - the target Naumu space id. This is an identifier, not a credential: forks, mirrors, and screenshots leak nothing. Admission to the space is identity-based (a git-history whitelist, admin domain wildcards, or a member-approved request-to-join), so a leaked space id grants no access.
- `slug` - the space's URL slug (the segment after `/spaces/` on naumu.ai). Only used to compose the `Naumu-Thread:` commit trailer without an extra `naumu_list_graphs` call; the setup knows it when it writes the file.
- `tracking.topic` - the human-readable name of the topic work-log entries are filed under, kebab-case. `work-log` is what the Naumu team uses, but pick whatever name fits your space. Omit the whole `tracking` block to opt out of the write half and keep retrieval only.
- `tracking.topicId` - the id of that topic (`topic-...`, from `naumu_list_topics`). This is the field the agent actually passes as `topicIds` on the first `naumu_delegate` call: `naumu_delegate` files by id, never by name, and a thread created without `topicIds` lands in the space-wide `#misc` feed and can never be re-filed. The quickstart prompt creates the topic during setup and writes both fields. If an older `.naumu` has only `topic`, the skill resolves the id once via `naumu_list_topics` - add `topicId` to save every session that call.
- `tracking.commitTrailer` - when `true` (the default), commits for a tracked task carry a `Naumu-Thread:` trailer linking the work-log thread, so `git blame` -> commit -> trailer leads a future teammate to the original reasoning.

### AGENTS.md section (from `templates/AGENTS-section.md`)

A short MANDATORY rule appended to the repo's `AGENTS.md`. AGENTS.md is the one file every coding harness reads unprompted, so it is the discovery vector. It is written as an imperative with the literal call shape, not a description: a descriptive sentence ("record the work back when done") was verified to never fire on a real customer repo - 27 threads, zero filed, zero commit trailers. The rule names three trigger points (start of a task, milestones such as decisions, pivots and blockers, and done), spells out `naumu_delegate` with `topicIds: [tracking.topicId]` on the first call and `threadId` on the rest, the `Naumu-Thread:` trailer, and the offer-once-never-auto-run setup. A harness that reads AGENTS.md without the skill installed still knows exactly what to call.

## Conventions in this registry

- Team-framed copy: present Naumu as the team's tooling convention, not a vendor funnel. No marketing hype, no exclamation-heavy tone.
- Hyphens, never em dashes, in any developer-facing string.
- Consent stays with the human: the skill offers, it never auto-runs an install or a commit.
- The skill is versioned here and installed per developer, so updates ship without touching the N customer repos that only committed `.naumu` + the pointer.

## Vendoring (air-gapped teams)

Teams that cannot install from a public registry may vendor `skills/naumu/SKILL.md` (and `templates/` for the committed artifacts) into their own repo. The AGENTS.md pointer works either way - point it at the vendored path instead of the `npx skills add` command.
