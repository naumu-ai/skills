---
name: naumu-import
description: Import a large unstructured corpus into your team's existing Naumu space over the Naumu MCP - doc dumps, exports, ticket archives with comment chains, piles of images, PDFs, audio and video. Triggers on `/naumu-import <path>`, `/naumu-import --resume`, and on plain asks like "import this export into Naumu", "bring these files into our space", "get this archive into the team graph". It imports INTO a space that already exists and never creates one - if the repo has no space or no Naumu MCP connected yet, use the `naumu` skill's connect flow first. It is a one-time import, not a live sync: Naumu's built-in integrations keep issue trackers and chat mirrored.
---

# Naumu import

Take a pile of files and land it in the team's Naumu space as something people can actually search: long documents kept verbatim as notes, media uploaded and embedded where it belongs, tickets and their comment history preserved, and - when the team wants it - typed nodes with real edges over the top.

The space already exists. This skill extends what the team built - it adds to the schema where the corpus needs types that are missing, dedupes against content that is already there, and never reshuffles work it did not create.

Runs are long. Everything is checkpointed to a run directory on disk, so a killed session resumes without duplicating a node or re-uploading a byte.

## When this fires

- `/naumu-import <path>` - import the corpus rooted at `<path>`.
- `/naumu-import --resume [runDir]` - continue the most recent unfinished run, or the one named.
- A plain ask that means the same thing: "import this export into Naumu", "bring these files into our space", "load this archive into the team graph".

Flags:

- `--dry-run` - run Phase 0 through Phase 6 and stop at the plan. Zero writes, including the Phase 0 probe and Phase 5's schema registration - additions are drafted and shown, not registered; capabilities are recorded as `unprobed` and the plan says which items depend on them. A dry run **never asks for consent and never makes the fan-out offer**: it presents the same six-part summary as a plan and exits. A dry run confers no consent on a later real run - that run asks at its own gate.
- `--state-dir <dir>` - put the run directory somewhere other than `./.naumu-import/`.

Preconditions: a Naumu MCP server is connected and a target space exists. If either is missing, stop and point at the `naumu` skill - it owns connecting a repo and joining a space. Do not improvise setup here.

The run also needs three things from the harness itself: **local filesystem writes** (the run directory and `staging/` are the run's only durable memory), **shell execution** (Phase 1's survey walk and Phase 2's transform scripts), and **an HTTP client that can perform fixed-length PUTs with custom headers** (the presigned upload path). Check all three before Phase 1. A harness that discovers a missing one mid-run has already spent the consent gate.

## What this is not

- **Not a live sync.** This is a one-time import. Keeping an issue tracker or chat tool mirrored is what Naumu's built-in integrations are for.
- **Not a space creator.** The target always pre-exists. Never call a graph-creation tool, never "start a fresh space to be safe".
- **Not a cleanup pass.** It never bulk-deletes, never reparents pre-existing children, never renames the team's types. Problems it did not cause get reported, not fixed.

## Parent edges and mesh edges

Two words carry the whole of Phases 7 and 9, so get them straight before the first edge.

Every node hangs from exactly one **parent edge** - created by passing `isParent: true` to `naumu_add_edge` - and those edges together form a single rooted hierarchy under the space's existing root. Every other relation is a **mesh edge** (`isParent` omitted, which is the default): it carries meaning but no hierarchy, and a node may have any number of them.

`naumu_list_dense_nodes` counts only parent edges, and `naumu_reparent` manages only parent edges. An import that creates edges without ever setting `isParent` builds a graph with no hierarchy at all - hub detection returns nothing, every node reads as an island, and Phase 9 passes by finding nothing to do.

## The consent gate

**One mandatory stop, and it closes the last read-only phase**: Phase 4 Schema fit in graph mode, Phase 3 Triage in artifacts mode. Everything before it is read-only reconnaissance, with one disclosed exception: the Phase 0 capability probe writes a single throwaway note containing no corpus content.

Nothing else touches the space until the human says yes: no schema change, no node, no note, no upload.

Show, in one compact message:

1. **The target space and the identity writing to it** - space name and id, how it was resolved (named by you / from the repo's `.naumu` file / picked from the list), and whether this connection is authenticated as a user or as a bot/API key, which is what decides whether topic filing is available at all.
2. **The destination** - which topic the import files under or that it is space-wide, whether that topic was verified closed, and whether this run is artifacts only or artifacts plus graph. For a gated import, say plainly that graph nodes are visible to every space member and that closed topics are team-level privacy, not isolation. For an artifacts-only run, add one line saying that once the run closes you will offer, once, to extract the concepts and enrich the graph as a follow-up pass - so the plan is honest about what comes next rather than surprising the user with it at the end.
3. **The corpus** - file count, total bytes, the main kinds, the largest items, and how many logical items each staging bucket holds.
4. **The plan** - how many items land in each tier, how many new types the schema needs and their names (graph mode), roughly how many calls and how long, and what gets deferred or skipped and why.
5. **What leaves the machine** - plainly: "N files totalling X GB will be uploaded to your team's Naumu storage and will be visible to <everyone in the space | members of #topic>." Name the probe note so nobody wonders what it is.
6. **The run directory path** and the exact resume command.

Then stop and wait. Do not start on a maybe, do not re-ask a second time in the same session, and do not narrow the question down to "shall I just do the small files first". If the answer is no, write `report.md` with the plan that was declined and leave the space untouched.

Everything after the gate proceeds without further confirmation, with three named exceptions: the optional fan-out offer in Phase 6, the enrichment offer an artifacts-only run makes after Phase 10 has closed, and - if that offer is accepted - the schema gate the enrichment pass runs at the close of its own Phase 4.

The byte total is an estimate, not a budget the run can quietly overrun. If the actual uploads would exceed it by more than 20%, pause, say by how much and why, and ask again before continuing. Record the corrected figure and the fresh consent in `run.json`.

## The run directory

Every run owns a directory: `./.naumu-import/<corpus-slug>-<YYYYMMDD-HHMM>/`, overridable with `--state-dir`.

`.gitignore` handling depends on where that directory lands. **Inside a git work tree**: for a space-wide run, offer once to add `.naumu-import/` to `.gitignore`; in gated mode do not offer, write the `.gitignore` line **before** the first staged file. Staging materializes gated content in plaintext, and that must not become committable. **Outside a work tree** (`~/Downloads/export/` and the like): skip `.gitignore` entirely - there is nothing to ignore into. Either way, say at the consent gate that staging materializes content in plaintext on disk, and where.

| File | Holds |
|------|-------|
| `run.json` | mutable header: runId, corpusRoot, graphId, identityKind (user/bot), destination (topic + graph/artifacts), execution (sequential/fan-out), planTier, capabilities, current phase, totals, consent, enrichment (the post-import offer and its answer), lastHeartbeat |
| `manifest.jsonl` | one line per raw file and one per staged logical item (last line wins per id) |
| `staging/` | derived, disposable normalized content, bucketed by kind |
| `status.jsonl` | item transitions and the artifact ids each item produced |
| `todo.md` | a regenerated human checklist per bucket - a view of the ledgers, never read back as state |
| `sibling-map.jsonl` | graph mode: dedup keys (lowercased label plus type) mapped to node ids, each carrying an `origin` that says how the id got there, plus an `intent` row written before every add; append-only, first write wins |
| `uploads.jsonl` | one row per upload, staged `presigned` / `put_ok` / `bound` / `failed` |
| `merges.jsonl` | graph mode: one line per Phase 8 collision merge, staged `queued` / `content-merged` / `edges-repointed` / `edges-removed` / `node-removed` |
| `schema.json` | graph mode: the merged schema, written at the Phase 5 gate and rewritten only by mid-run schema evolution |
| `trace.md` | one line per event: `[HH:MM] phase <n> <event> - <detail>` |
| `report.md` | the closing account, written in Phase 10 |
| `locks/` | `run.lock` on every run, written when the directory is created; shard claims in fan-out |
| `shards/` | fan-out only |

Every ledger is append-only. **Read `references/state.md` before writing the first ledger line, and again at the start of every resume** - it carries the exact record schemas, the item lifecycle, the upload and merge stage tables, and the resume protocol.

## Phases at a glance

| # | Phase | Read | Closing gate |
|---|-------|------|--------------|
| 0 | Preflight and destination | `references/state.md` | space resolved, destination settled, run dir written, capabilities probed |
| 1 | Survey | inline | `manifest.jsonl` covers every file, no bodies read |
| 2 | Transform | `references/transform.md` | logical items staged and appended to the manifest |
| 3 | Triage | `references/triage.md` | every item has a tier and a reason |
| 4 | Schema fit **(graph mode only)** | `references/schema.md` | additions drafted, **consent gate cleared** |
| 5 | Schema gate **(graph mode only, hard)** | `references/schema.md` | additions registered, `schema.json` frozen |
| 6 | Plan the run | `references/fanout.md` if fan-out is accepted | order fixed, sibling map pre-seeded |
| 7 | Ingest | `references/ingest.md`, plus `attachments.md` / `tickets.md` when those tiers exist, plus `references/restructure.md` in graph mode | every item terminal in `status.jsonl` |
| 8 | Reconcile | `references/ingest.md`, `references/state.md`; `references/fanout.md` only if the run sharded | collisions merged, uploads swept |
| 9 | Restructure **(graph mode only)** | `references/restructure.md` | shape passes are idempotent on a re-run |
| 10 | Close | inline | `report.md` written, DONE line traced |

In artifacts mode, phases 4, 5 and 9 do not run at all: no schema is read for fitting, no schema is written, no nodes of any kind are created, and the graph is never restructured. They run later only if the user accepts the enrichment offer, which flips the run to graph mode and re-enters at Phase 3 - see **Post-import graph enrichment**.

Work the phases in order. Do not narrate every step in chat - the trace file is the detailed channel. Chat gets the Phase 0 line, the consent gate, milestone lines, and the close.

---

## Phase 0 - Preflight and destination

### Resolve the target space (never create one)

In this order, first hit wins:

1. **Explicit user instruction** - a space id or slug named in the invocation or the surrounding request.
2. **A `.naumu` file at the repo root** - use its `space` field.
3. **Ask.** Call `naumu_list_graphs` and put the list to the user. Never pick for them, never default to a personal space.

Confirm you are authenticated with `naumu_whoami`. If no space resolves, or no Naumu MCP is connected, stop and point at the `naumu` skill. Never create a graph; the target always pre-exists.

### Read the identity kind before asking anything else

`naumu_whoami` also says **what** you are authenticated as. Record it as `run.json.identityKind`: `user` for an OAuth or session identity, `bot` for an API key - which is how the stdio `@naumu/mcp` server authenticates.

**Check the server is new enough at the same moment**, per the "Server version and staleness" section below: `naumu_create_topic` in the tool list and a `noteId` destination on `naumu_request_attachment_upload`. If either marker is missing the server predates `@naumu/mcp` 0.10.0 - say so here, before the destination question, and tell the user how to refresh.

A bot identity cannot file into a topic. `naumu_create_note` rejects `topicIds` from bots, because bots hold no topic membership, so a gated import on this connection dies at the first note - after consent, with the schema already extended. Say so **before** the destination question, not after it, and offer exactly two ways forward: run space-wide on this connection, or reconnect with a user credential and file into a topic. Do not present the topic list as if it were available and do not plan a gated run you cannot execute.

### Ask where the import belongs (one question, two parts)

Every import asks this, once, in a single gathering moment - before the survey, and never split across two turns:

1. **Which topic should this file under?** Call `naumu_list_topics` and offer the list. "None - space-wide" is a valid answer.
2. **Artifacts only, or artifacts plus graph?** **Artifacts only is the default and what you recommend.** Put the trade-off plainly and let the team choose:

   - **Artifacts only (default).** Notes and media, nothing added to the graph. It is the fastest path from a pile of files to material the team can open and search - and an agent can already reason over the imported artifacts and search across them the moment they land, so nothing useful is waiting on the graph.
   - **Artifacts plus graph.** The same artifacts, plus typed nodes and edges extracted over them. It takes meaningfully longer, because every eligible item is read and its concepts pulled out, but the important concepts end up as typed nodes - which makes ongoing agent reasoning across the material more efficient once it is done.

   Say that the graph half is not a now-or-never choice: after an artifacts-only import closes, this skill offers once to extract the key concepts and enrich the graph as a follow-up pass over the same run, with a duration estimate scaled from what the artifacts pass actually cost. See **Post-import graph enrichment**. Choosing the fast path locks nothing in.

   The topic half IS permanent. Content cannot be re-homed over MCP afterwards, so a wrong topic - or a wrong space-wide - stands for the whole run, enrichment pass included.

When even titles are sensitive, or when the team wants the material present without weaving it into their graph, **artifacts only is the right call regardless of what the graph pass would buy them**. Nodes are space-communal and no topic hides them, so that is a judgment about the content, not about time.

If the request already names a topic ("import the Zendesk dump into #traderevolution"), that half is answered - still ask the other half. Do not guess a sensitive corpus into space-wide.

Record the answer in `run.json.destination` and restate it at the consent gate.

### Every note carries an audience

The answer above decides one parameter that every single `naumu_create_note` call in this run must carry: a topic import passes the destination's `topicIds`, a space-wide import passes `sharedWithSpace: true`. Never neither.

A note created with neither does not come out private. On a user session it inherits the space's default visibility, which normally falls back to `internal` - readable by every member of the space. For a gated import that is a **leak**, not a stranded note: the agent believes it wrote something harmless and private, and it published gated bodies space-wide. Always pass one. Record the audience the create response reports on that item's first status line, so a resume can check what each note actually landed as rather than assuming.

### Gated destinations

Filing into a **closed** topic gates the import. Verify BOTH with `naumu_list_topics` before relying on it:

- `visibilityMode` is `closed`;
- your account is a member. Rely on `isMember` when `naumu_list_topics` returns it. When the field is absent, the closed topic's presence in your list **is** the membership proof - closed topics you are not in are omitted from the list entirely.

If either check fails, or you cannot verify both, STOP before the consent gate. `naumu_create_topic` ships on server builds >= 0.10, but it requires admin rights and is absent from some connectors - if the connected server exposes it and your account is an admin, it can create the closed topic (creating one adds the caller as a member, which settles both checks at once). Otherwise the user must create the closed topic in the app and add the importing account to it, then rerun. No MCP tool reconfigures an existing topic's visibility either way. Never fall back to space-wide for a corpus the user wanted gated.

A `default` or `open` topic files content without gating anything - every member reads it without joining. That is a fine destination, it is just not privacy, and saying so plainly beats letting the user assume otherwise.

Where the topic is closed, its `topicIds` are also the access boundary. The note is the gate: files embedded in it inherit its audience automatically, in lists, search, and byte downloads alike. Every `naumu_delegate` call carries the same `topicIds`.

Graph nodes are the exception, because they are space-communal by design and no topic or setting hides them. In graph mode with a gated topic, nodes carry labels, structural attributes and edges - enough to keep the graph useful - while bodies, transcripts, and client-identifying prose stay inside the gated notes. When even titles are sensitive, that is the case for artifacts mode.

Honesty that belongs in the consent message for a gated import: closed topics are team-level privacy, not isolation - Naumu platform operators can access content for support, and anything placed on the graph is visible to every space member. Truly confidential material that must not share a graph with the rest of the team belongs in its own space, not in a topic.

### Check for a resume, then create the run directory

If invoked with `--resume`, or if `./.naumu-import/` holds a run whose `run.json.phase` is anything but `done`, go to **Resuming an interrupted run** below before doing anything else. Never start a second run over the same corpus while one is unfinished.

Otherwise make `./.naumu-import/<corpus-slug>-<YYYYMMDD-HHMM>/` (slug derived from the corpus folder name) and write the initial `run.json`. Handle `.gitignore` per the run-directory rules above.

**Write `locks/run.lock` as soon as the directory exists** - on every run, sequential and fan-out alike, carrying your own identifier and a timestamp. Then, before Phase 1 starts, check for a live foreign `run.lock` over the same corpus: if another session holds one, stop and tell the user, rather than proceeding. Two sessions on one corpus both register schema additions and trip each other's Phase 5 integrity check, and neither can see the other's ledger.

### Probe capabilities and the plan ceiling (never halt on failure)

Servers differ by version. Find out what this one can do rather than assuming:

1. `naumu_create_note` a note titled `Naumu import probe <runId>`, carrying the run's audience like every other note it makes. Record its id as `run.json.capabilities.probeNoteId`: a resume re-probes by appending to that same note and never creates a second one.
2. `naumu_request_attachment_upload` against that `noteId` for an 11-byte text file. If the tool exposes no `noteId` parameter at all, that is the probe failing - take the same downgrade, and do not substitute the thread flow.
3. PUT the bytes to the returned URL.
4. `naumu_note_append` a sole-line `![probe](attachment://<id>)` to the probe note.

All four pass -> `capabilities.noteAttachments: true`. Any step fails -> **downgrade, do not halt**: media items are triaged `deferred` with reason `server-lacks-note-attachments`, the rest of the run proceeds normally, and `report.md` says plainly which files did not make it and why. A capability the server lacks is a smaller run, not a failed one. Record the outcome in `run.json.capabilities`, name it in the consent message, and name the probe note itself in both the consent message and `report.md` so nobody has to wonder what it is.

One capability costs no call at all: read the `naumu_create_note` schema this server shows you. If it advertises a `markdown` input, the server writes the body at create time and every note in this run lands in a single call carrying its content - record `capabilities.noteBodyOnCreate: true`. If the field is absent, record `false` and the run uses the older create-then-`naumu_note_append` sequence. Detect the field, never a version number.

Then probe the plan ceiling: call `naumu_request_attachment_upload` for a deliberately oversized `video/mp4` against the same note. No bytes are transferred - you only want the rejection, which names the ceiling and the plan. Record it as `run.json.planTier`. If it does not reject, record `planTier: unknown` and treat large media conservatively.

Two ceilings matter and they are different: agent-readable types (image, text, PDF, office) are capped at 10MB on **every** plan and upgrading does not lift it; the plan ceiling applies to video, audio and archives. Keep the two straight in every message you write about deferrals.

### Trace

```
[HH:MM] phase 0 preflight - space=<name> <id>, dest=<topic|space-wide> mode=<graph|artifacts>, runDir=<path>, noteAttachments=<yes|no>, noteBodyOnCreate=<yes|no>, planTier=<tier>
```

---

## Phase 1 - Survey

A deterministic walk of the corpus into `manifest.jsonl`. **No file bodies are read in this phase**, and the conversation's context cost must not scale with the corpus - a 4,000-file dump and a 40-file dump should cost the same to survey.

Per raw file, record: `id` (first 12 hex of the sha256 of the path relative to the corpus root), `path`, `bytes`, `mime`, `sha256`, `kind`. Tier and reason are filled in Phase 3.

Rules:

- Walk with the shell and stream straight to the ledger. Never load the listing into the conversation.
- Classify `kind` from extension and mime detection plus size. Opening files to decide belongs to Phase 2.
- Hash content fully under 50MB. Above that, hash the first and last megabyte plus the byte length and mark the record as a sampled hash.
- Follow no symlinks out of the corpus root. Record but do not descend into archives.
- Only aggregates enter chat: counts and bytes per kind, the ten largest files, the directory tree two levels deep.

If the walk exceeds roughly 50,000 files, say so, report the aggregate shape, and ask whether to import a subtree instead. A corpus that big usually wants scoping, not brute force.

```
[HH:MM] phase 1 survey - 4,812 files, 6.2 GB, kinds: text 611, image 1,902, pdf 288, video 44, other 1,967
```

---

## Phase 2 - Transform

Raw files are not the unit of work. A browser-saved article is twenty files; a support ticket is a JSON plus a folder of attachment siblings. Transform turns raw files into **logical items** - one article, one ticket, one document - and materializes a parse-friendly copy of each under `staging/`. From Phase 3 on, the logical item is the unit of work, status, resume, and fan-out sharding.

Bulk normalization is script work: write a deterministic script into the run directory and run it. Corpus bodies do not pass through your context - calibrate the script on a bounded sample per bucket and verify its output the same way.

**Read `references/transform.md` now.**

Close the phase when every logical item is staged and appended to the manifest.

```
[HH:MM] phase 2 transform - 812 articles, 178 tickets, 96 docs staged from 4,812 raw files; 2,914 noise skipped, 3 unparseable
```

---

## Phase 3 - Triage

Assign exactly one tier per item, first match wins: `ticket`, `defer`, `graph-extract`, `note-only`, `attach-caption`, `attach-only`, `skip`. The tier decides what gets read locally, what bytes leave the machine, and where the result lands.

**Read `references/triage.md` now** for the rubric, the reason codes, and the two ceilings.

In **artifacts mode**, this is the last read-only phase: close it by running the consent gate. Nothing is written to the space before the human says yes.

In graph mode, close it when every item holds a tier and a reason.

```
[HH:MM] phase 3 triage - graph-extract 611, note-only 288, attach-caption 1,902, attach-only 44, ticket 178, defer 37
```

---

## Phase 4 - Schema fit (graph mode only)

Call `naumu_get_schema` on the target **first**. Map the corpus onto the types that already exist; design additions only where genuinely nothing fits.

- Never rename, remove or repurpose an existing type. The space's existing root stands - do not impose a root of your own.
- Additions carry the same discipline as any Naumu schema: no umbrella types, a `kind` enum only when its values are facets of one coherent dimension, a polymorphic parent for people.
- An addition needs a one-sentence definition with no "or" in it. If you cannot write that sentence, it is a catch-all and it must be decomposed before Phase 5.

**Read `references/schema.md` now.**

When the additions are drafted, **run the consent gate**. Do not register a single type before the human says yes.

```
[HH:MM] phase 4 schema-fit - reused 9 existing types, proposing 3 additions: Incident, RunbookDoc, ExternalVendor
```

---

## Phase 5 - Schema gate (graph mode only, hard)

Register the approved additions, then verify. This gate is not advisory.

**Register with the incremental tools**: `naumu_add_node_type`, `naumu_add_connection`, `naumu_add_attribute`. Prefer them in an existing space. `naumu_update_schema` writes a full schema definition - if you use it at all, it must restate every pre-existing type verbatim, or you will silently remove the team's work.

Then call `naumu_get_schema` again. All of these must hold, and `references/schema.md` carries the exact procedure and the audit format:

1. Every type you added is present, and declares a parent connection into a type that already existed or that you added.
2. Every pre-existing type is still present and unchanged. If one is missing or altered, you broke something - halt, say exactly what changed, and do not continue.
3. The kind-enum audit passes for every added type with a `kind` attribute of three or more values, with the audit written to the trace before the gate can pass. Three or more distinct natural parents must split; so must values that name fundamentally different kinds of facts even under one parent.
4. No added type carries a banned umbrella name (`Topic`, `Concept`, `Note`, `Item`, `Thing`, `Entry`, `Element`). Pre-existing types keep their names; this rule constrains only what you add.

**Freeze the schema.** Write the merged result - pre-existing types plus additions - to `schema.json`. Phase 7's edge-direction gate, any fan-out worker, and every resume read that file. They never re-derive it.

On failure: `[HH:MM] phase 5 HALT - <what failed>` and stop. Surface it to the user; a bad schema poisons every node that follows.

```
[HH:MM] phase 5 gate - 12 types (9 pre-existing intact, 3 added), kind-audit clean, schema.json frozen
```

---

## Phase 6 - Plan the run

- **Order** items so entities are established before the things that reference them: canonical documents first, long-tail second, tickets after the people and projects they name, loose media last (it attaches to notes that must already exist).
- **Pre-seed the sibling map** from the space's existing content, in graph mode. Call `naumu_filter({graphId, nodeTypes: [<type>], limit: 200, sortBy: "updatedAt"})` per target type and append each result as an `origin: "preseed"` row in `sibling-map.jsonl`. **Write both arguments out every time**: the tool defaults to `limit: 50` sorted by `sortKey`, so an unadorned call seeds a quarter of what you think it does, in an order that has nothing to do with recency. 200 is the cap and it does not paginate, so a larger type is seeded from its 200 most recent; record seeded-versus-total per type in `run.json` and name every truncated type in `report.md` as elevated duplicate risk. Skip any result id already present in `sibling-map.jsonl` - on a resume the map already holds this run's own creations, and re-seeding them as `preseed` would relabel the run's own nodes as the team's. Skip the seed entirely and dedup only sees this run's own work - you will duplicate everything the team already had. In artifacts mode there is nothing to pre-seed.
- **Record the plan** in `run.json`: items per tier, bytes to upload, estimated calls, the order. Turn calls into a duration honestly: an API-key (bot) session is rate-limited to 60 requests per minute for the whole account, so estimate against a ceiling of roughly one call per second and say the estimate is a floor. An OAuth user session is not held to that limit and runs as fast as the server answers.
- **Fan-out is optional and off by default.** Offer it only when the manifest exceeds roughly 400 logical items or 150MB of `graph-extract` text, only if this harness actually has subagents, and only on an explicit yes. **Read `references/fanout.md`** before accepting.

```
[HH:MM] phase 6 plan - 1,086 logical items in 3 passes, 6.1 GB to upload, sibling map pre-seeded with 412 existing nodes, execution=sequential
```

---

## Phase 7 - Ingest

The long phase. Everything else exists to make this one resumable. Work the items in the Phase 6 order, writing `in_progress` before an item's first write to the space and `done` after its last.

- **Gated mode.** Every `naumu_create_note` call carries `run.json.destination.topicIds`, and every `naumu_delegate` call does too. Never put gated bodies, transcripts, or client names into node content or attributes; nodes are visible to the whole space.
- **Artifacts mode.** Nothing calls `naumu_add_node` or `naumu_add_edge` at all - triage has already routed every item to a note, and media embeds into the note that owns it.
- **Bulk copy-through may be scripted.** When note bodies are faithful copies of staged content, a run-directory script may write them - same audience rules, same ledger rows, samples verified. `references/ingest.md` owns the conditions.

**Read `references/ingest.md` now.** Also read `references/attachments.md` if any item is `attach-caption`, `attach-only`, or a `note-only` item that embeds a file, and `references/tickets.md` if any item is `ticket`. In graph mode **read `references/restructure.md` now as well**: the ingest loop invokes a mini-restructure pass mid-phase and that file carries it. Do not wait for Phase 9 to load it.

That mini pass runs **every 25 items, or whenever more than 100 nodes have been added since the last pass, whichever comes first** - plus one final pass at the last item of the run, so nothing is left unaudited. Not once per item. A corpus of a thousand items would otherwise spend a thousand dense-node scans on a graph that barely moved between them. In fan-out the mini pass is off entirely; the coordinator's Phase 9 does all of it.

Close the phase when every item holds a terminal status in `status.jsonl`.

```
[HH:MM] phase 7 item L_4c81a09e chunk 2 - 6 added, 3 merged, 11 edges, 2 uploads bound
```

---

## Phase 8 - Reconcile

Fold the ledgers, merge what raced, and finish what never finished. A run that skips reconcile leaves duplicate entities and orphaned uploads behind for the team to trip over.

In **artifacts mode** this phase shrinks to three things: every item terminal, every note accounted for, uploads swept. There are no nodes, so there are no collisions and no near-miss sweep.

**Read `references/ingest.md`** - it owns the five-step merge protocol, which applies in sequential mode too, because a long single-threaded run hits the same races across chunks - and **`references/state.md`** for the fold rules. Read `references/fanout.md` only if the run sharded.

Close the phase when every item is terminal, every collision merged, and every bindable upload bound.

```
[HH:MM] phase 8 reconcile - 1,086 items terminal, 6 collisions merged, 2 near-miss merges, 3 ambiguous reported, 1 orphan upload bound
```

---

## Phase 9 - Restructure (graph mode only)

Shape, not meaning. Reparent only - never delete a node, never rewrite content, and never move a pre-existing child the import did not create. Coordinator only, single-threaded, after reconcile has finished.

**Read `references/restructure.md` now.**

Close the phase when running it again on the settled graph would be a no-op.

```
[HH:MM] phase 9 restructure - 4 hubs split, 9 mini-hubs created, 214 nodes reparented, 2 islands collapsed, 0 pre-existing nodes touched
```

---

## Phase 10 - Close

1. **Verify** every item is terminal and every `bound`-able upload is bound. Regenerate `todo.md` one last time; it should read as complete.
2. **Write `report.md`.** It is the honest account, and it is the thing a teammate reads six months from now:
   - what landed: notes, uploads, and in graph mode nodes, edges and which types are new;
   - what was deferred, grouped by reason, each labelled liftable by a plan upgrade or not liftable at all;
   - what was skipped and why (noise, empty, duplicate bytes, unparseable);
   - what failed, with the error;
   - shape problems observed in the pre-existing graph and deliberately left alone;
   - the run directory path and the exact resume command.
3. **Write the tooling-feedback block** into the trace before the DONE line: errors and unexpected behavior, descriptions that did not match what a tool did, workarounds you applied, what would have made the run faster or better. Write "none" for an axis that genuinely did not apply. Be specific enough to act on.
4. **Record the run in the space** if the team keeps a work log: one `naumu_delegate` call with a short bulleted summary. Bullets, not prose - a human scrolls these. In gated mode the call carries the gated `topicIds`, and the summary names no gated content - a space-wide record of a gated import is itself a leak.
5. **Tell the user in chat**, briefly: the space, the destination, the counts, what was deferred and whether a plan upgrade would change that, and the run directory path.
6. **If `destination.mode` is `artifacts`, make the enrichment offer** - once, in the same message or immediately after it, per **Post-import graph enrichment** below. A graph-mode run makes no offer.

```
[HH:MM] phase 10 DONE - 3,912 nodes, 7,104 edges, 288 notes, 1,946 uploads, 37 deferred, 630 skipped, 2 failed, 4h12m
```

---

## Post-import graph enrichment

Only for a run whose `destination.mode` is `artifacts`, only once Phase 10 has closed cleanly, and only once. It is not a new import - it is the same run, continued in graph mode over the same ledgers.

### The offer

In chat, right after the closing summary: everything is imported and verified, and the material is already searchable - would the team like the key concepts extracted and the graph enriched now?

The offer carries an honest duration illustration, and **both halves of it are derived from this run's own numbers**. Do not quote any figure written here; there is none to quote.

- **How many items would be eligible.** Fold the manifest: the `note-only` items that artifacts mode demoted out of `graph-extract`, plus the `ticket` items. Those are what extraction would read. Say the count.
- **Roughly how long.** Scale it from what the artifacts pass actually cost on this corpus and this connection. Take the wall-clock Phase 7 spent per item, multiply by the eligible count, and multiply again by an honest read-and-extract factor - extraction reads bodies and writes nodes and edges where the artifacts pass mostly moved bytes, so it is several times the per-item cost, not a fraction more. Quote a range, say it is scaled from this run's own timings, and say it is a floor on a bot identity, which the server caps at 60 requests per minute.

Record the offer in `run.json.enrichment`: `offered` and `offeredAt`, the `eligibleItems` count and the `estimateShown` range you actually quoted, then `decision` and `decidedAt` once the user answers. `state.md` owns the shape.

**Do not re-ask in the same session if the answer is no.** Write `decision: "declined"`, say the run directory keeps everything needed to do it later, and stop. A later session reads that field and leaves it alone unless the user raises it.

```
[HH:MM] phase 10 enrichment-offer - 874 items eligible, estimate 3-5h, answer=accepted
```

### On yes: the same run, continued in graph mode

1. **Flip the mode.** Set `destination.mode` to `"graph"`, set `destination.enrichmentOf` to `"artifacts"` - the mode the run closed in - and `destination.modeChangedAt`, then set `phase` back to `3`. That marker is what tells any later resume it is inside an enrichment pass rather than a fresh graph run. Field shapes live in `references/state.md`. Trace it:

   ```
   [HH:MM] phase 3 enrichment-start - mode artifacts -> graph, enrichmentOf=artifacts, re-triage of 874 eligible items
   ```

2. **Re-triage under the graph rubric** (Phase 3, `references/triage.md`). Items already terminal as artifacts **keep every artifact they produced**. The re-triage decides one thing: which items are `graph-extract` eligible now that the artifacts-mode demotion no longer applies.
3. **Phase 4 and Phase 5 run for the first time.** Schema fit against the live schema, then the hard gate and `schema.json`. The consent gate at the close of Phase 4 runs, **scoped honestly to what is actually being decided**: the bytes are already uploaded and the notes already exist, so this gate is about the schema additions and the node and edge estimate, not about what leaves the machine. Say that plainly rather than re-showing the byte total as if it were still pending. Record the answer as `run.json.enrichment.schemaConsent`; the top-level `consent` covers the artifacts pass and is never rewritten.
4. **Phase 6 plans as usual**, sibling-map pre-seed included. This run has never seeded - artifacts mode writes no sibling map - so the pre-seed is a first seed, with the same `limit: 200, sortBy: "updatedAt"` discipline.
5. **Phases 7, 8 and 9 run as any graph run does**, extracting over the staged content of the eligible items, reconciling, and restructuring.
6. **A fresh Phase 10 close APPENDS to `report.md`** under its own dated heading, with its own counts. It never overwrites the artifacts pass's account - that account is what a teammate reads to understand what the notes are.

### Invariants

- **Never delete, re-create, or re-upload an existing artifact.** Every note and every bound upload from the artifacts pass stands exactly as it is; `uploads.jsonl` folds to `bound` and stays there.
- **The created notes stay the canonical bodies.** Extraction links nodes to them - the note id rides on the Document node as the attribute `schema.json` registered for it, per `references/ingest.md` - and never copies a body into node content. A `note-only` item that already has its note gains its Document node and nothing else.
- **A killed enrichment pass resumes exactly like any other resume**, under `references/state.md`'s protocol. The recorded `enrichmentOf` and mode flip are what a fresh session reads to know where it is; no resume rule changes, including "never redo phases 4 and 5" once the enrichment pass has cleared its own schema gate.

---

## Resuming an interrupted run

Written for a session that has no memory of the original run.

**1. Find the run.** With `--resume <runDir>`, use that path. Otherwise take the most recent `./.naumu-import/*/` whose `run.json.phase` is not `done`, showing them and asking which if several are unfinished. If none exists, this is a fresh import - go to Phase 0.

**2. Read `references/state.md` in full.** It is the authority on the ledger formats, the fold, the per-tier recovery table, and the whole resume protocol. Do not reconstruct any of it from what the files look like.

Three invariants hold whatever the protocol says next:

- **Never re-target a different space and never substitute a moved corpus.** Stop and ask.
- **Never redo phases 4 and 5.** They ran once, the gate passed, and `schema.json` stands as written. A resume verifies it, never rewrites it, and never re-opens schema design - that would only fight the graph it already built.
- **Partial items first.** Every item at `in_progress` is known to be partially applied. Handle those before any pending work, in verify-then-continue mode.

Then carry on through the remaining phases as normal, and close with Phase 10.

## Server version and staleness

This skill requires `@naumu/mcp` **0.10.0 or newer**. Do not trust `serverInfo.version` - older servers report a stale constant. Check for the two 0.10.0 markers instead: `naumu_create_topic` present in the connected server's tool list, and a `noteId` destination on `naumu_request_attachment_upload`. Either one missing means the running server predates what this skill needs - a run on it cannot file into a topic it creates and cannot bind media to notes at all.

Say so **at Phase 0, before the destination question**, not after the corpus has been surveyed, and tell the user how to refresh:

- **Remote connection** (`https://naumu.ai/api/mcp`) - reconnect: `/mcp` in Claude Code, or restart the session.
- **Local `@naumu/mcp` binary** - `npm i -g @naumu/mcp@latest`, or clear the `npx` cache, then restart the session.

Newer conveniences are a different question and are **never version-gated**: the `markdown` field on `naumu_create_note` is feature-detected by Phase 0's capabilities probe, recorded as `capabilities.noteBodyOnCreate`, and re-detected on every resume. That doctrine is authoritative - detect the field, never a version number.

A stale server discovered mid-run does not abort the run. It follows the existing capability-downgrade rules: defer the affected items with their reason, carry on with the rest, and name what did not land in `report.md`.

## Tool reference

Use only the tools the connected server exposes; resolve by capability if your harness prefixes names differently.

| Need | Tool |
|------|------|
| Identity, spaces, topics | `naumu_whoami`, `naumu_list_graphs`, `naumu_list_topics` |
| Read the target schema | `naumu_get_schema` |
| Add to the schema | `naumu_add_node_type`, `naumu_add_connection`, `naumu_add_attribute` |
| Write a whole schema definition (last resort) | `naumu_update_schema` |
| Create graph content | `naumu_add_node`, `naumu_add_edge` (pass `isParent: true` for the one parent edge; omit it for mesh edges) |
| Enrich a node that already exists | `naumu_update_node` |
| Dedup and lookup | `naumu_search`, `naumu_filter`, `naumu_get_node`, `naumu_list_node_connections` |
| Find shape problems | `naumu_list_dense_nodes` |
| Restructure | `naumu_batch_reparent`, `naumu_reparent` |
| Undo a mistake this run made | `naumu_remove_edge`, `naumu_remove_edges_bulk`, `naumu_remove_node` |
| Notes | `naumu_create_note` - pass `markdown` and the note lands with its whole body in that one call, so staged content goes over verbatim instead of being appended (and retyped) afterwards; where the schema has no `markdown` field, create then `naumu_note_append`. Then `naumu_note_read`, `naumu_note_append`, `naumu_note_insert`, `naumu_note_replace_section`, `naumu_note_replace` |
| Upload a file | `naumu_request_attachment_upload` |
| Post into an existing thread | `naumu_post_message` |
| Hand work to the space's agent | `naumu_delegate` |

## Failure handling

Each phase owns the failures that happen inside it: a single failed call by Phase 7, a failed probe by Phase 0, an unparseable source by Phase 2, a rejected upload by triage's caps, a broken schema by Phase 5. Two rules belong to the run as a whole:

| Situation | Do |
|-----------|-----|
| A call returns 429 (rate limited) | Back off and continue. **Never abort on a 429.** The stdio API-key server allows 60 requests per minute per account - shared by every session and every worker on that key - so a fast stretch of small writes hits it routinely. Wait, retry the same call, and keep going; a 429 is the server pacing you, not refusing you. |
| The same call fails three times in a row on different items | Stop, write the phase and the error to `run.json`, and tell the user. This is a server or auth problem, not a data problem. **429s do not count toward this rule** - do not let a rate-limit storm read as an auth failure and abort a healthy import. |
| The user interrupts | The ledgers are already on disk. Say the run directory path and the exact resume command, and stop cleanly. |

Never "clean up" a partial run by deleting what landed. The space is shared and other people may already be reading it. The ledger records exactly what was written and the report owns up to it; that is the safety net, not a rollback.

## Copy guidelines

- Team-framed: this is the team's import into the team's space, not a vendor flow.
- Hyphens, never em dashes.
- Honest about limits: say what was deferred, say whether a plan upgrade would change it, and never quietly drop a file.
