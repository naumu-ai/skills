# skills-registry

Source for the public `naumu-ai/skills` registry. What lives here is exported to that GitHub repo and installed per developer with:

```bash
npx skills add naumu-ai/skills
```

That install ships one skill, `naumu`. The skill body is intentionally generic and public - no hardcoded team, space, or company. A repo opts in by committing two small artifacts (see `templates/`); the skill reads them at runtime.

## The skill

- **`naumu`** (daily loop + cold-connect) - the behavior in a repo that has a `.naumu` file. It runs the pre-task retrieval reflex, writes the work-log + `Naumu-Thread` commit trailer, and - in a cold clone with no MCP connected - makes the one-time offer to connect (per-harness MCP add, `naumu_resolve_admission`, seat-limit relay, decline marker). It does not create spaces or write `.naumu`.

First-time setup (create/select a space, whitelist teammates from git history, write `.naumu` + the AGENTS.md section, open a PR) is not a skill - it is driven by the pasteable quickstart prompt (`QUICKSTART.md`), which the app also surfaces as a copy block on the Linked apps settings page. That keeps the on-disk skill purely about the day-to-day loop.

## Layout

```
skills-registry/
  README.md                          this file
  QUICKSTART.md                      the pasteable first-time-setup prompt (shipped in the app too)
  skills/naumu/SKILL.md              the daily skill (read+write loop, cold-connect etiquette)
  templates/naumu.json               the .naumu config template a repo commits
  templates/AGENTS-section.md        the ~4-line AGENTS.md pointer a repo commits
```

## How the loop closes

1. A developer installs the skill once (`npx skills add naumu-ai/skills`).
2. In a repo they own, they run the quickstart prompt (from the Linked apps settings page). It creates or selects a space, whitelists teammates from git history, writes `.naumu` + the AGENTS.md section, and opens a PR.
3. The PR merges. Now every other teammate's harness reads the committed AGENTS.md pointer, which tells it to install the same skill; the daily `naumu` skill offers (once) to connect.
4. Connected teammates get the team's memory before each task and leave a work-log trail behind them.

The repo artifact installs the skill; the quickstart prompt installs the repo artifact. That is the whole wedge.

## The committed artifacts

### `.naumu` (from `templates/naumu.json`)

JSON cannot carry comments, so the fields are documented here:

```json
{
  "space": "<space-id>",
  "track": {
    "workLog": true,
    "commitTrailer": true
  }
}
```

- `space` - the target Naumu space id. This is an identifier, not a credential: forks, mirrors, and screenshots leak nothing. Admission to the space is identity-based (a git-history whitelist, admin domain wildcards, or a member-approved request-to-join), so a leaked space id grants no access.
- `track.workLog` - when `true`, the agent records a short work-log entry into the space on task completion. Set `false` to opt out of the write half and keep retrieval only.
- `track.commitTrailer` - when `true`, commits for a tracked task carry a `Naumu-Thread:` trailer linking the work-log thread, so `git blame` -> commit -> trailer leads a future teammate to the original reasoning.

### AGENTS.md section (from `templates/AGENTS-section.md`)

Around four lines appended to the repo's `AGENTS.md`. AGENTS.md is the one file every coding harness reads unprompted, so it is the discovery vector. It tells agents to offer setup, never auto-execute, and points at `npx skills add naumu-ai/skills`.

## Conventions in this registry

- Team-framed copy: present Naumu as the team's tooling convention, not a vendor funnel. No marketing hype, no exclamation-heavy tone.
- Hyphens, never em dashes, in any developer-facing string.
- Consent stays with the human: the skill offers, it never auto-runs an install or a commit.
- The skill is versioned here and installed per developer, so updates ship without touching the N customer repos that only committed `.naumu` + the pointer.

## Vendoring (air-gapped teams)

Teams that cannot install from a public registry may vendor `skills/naumu/SKILL.md` (and `templates/` for the committed artifacts) into their own repo. The AGENTS.md pointer works either way - point it at the vendored path instead of the `npx skills add` command.
