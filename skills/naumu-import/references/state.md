# Run state: ledgers, lifecycle, resume

Read this in Phase 0, before the first ledger line, and again at the start of every resume. It is the authority on what each file holds and how a half-finished run is picked back up.

The run directory is the only durable memory the run has. If it is not in a ledger, it did not happen.

## Layout

```
./.naumu-import/<corpus-slug>-<YYYYMMDD-HHMM>/
  run.json            mutable header, rewritten atomically
  manifest.jsonl      one line per raw file, one per staged logical item
  staging/            derived, disposable normalized content, bucketed by kind
  status.jsonl        item transitions and artifact ids
  todo.md             regenerated human checklist, a view of the ledgers
  sibling-map.jsonl   graph mode: add intents and dedup key -> node id
  uploads.jsonl       one row per upload attempt
  merges.jsonl        graph mode: one row per Phase 8 collision merge stage
  schema.json         graph mode: the merged schema, frozen at the Phase 5 gate
  trace.md            the human-readable run log
  report.md           the closing account, written in Phase 10
  shards/             fan-out only: one file per worker shard
  locks/              run.lock on every run; advisory shard claims in fan-out
```

`--state-dir <dir>` moves the parent directory. Nothing else about the layout changes.

`staging/` is the one directory that is not state. It is derived from the raw corpus by Phase 2 and can be deleted and rebuilt at any time; `transform.md` owns its rules.

## Why append-only

Every `.jsonl` file is opened for append and never rewritten. A process killed mid-write leaves at most one torn trailing line, which the fold discards; everything before it is intact. Rewriting a ledger in place turns the same crash into a lost file.

`run.json` is the one exception, because it is a mutable header. Write `run.json.tmp` and rename it over `run.json`, so a reader never sees a half-written header. `todo.md` is regenerated wholesale each time, which is safe because nothing ever reads it back.

## run.json

```json
{"runId":"acme-export-20260804-1412","corpusRoot":"/Users/dev/dumps/acme-export","graphId":"7c1e9b40-2f55-4a0a-9d61-b8a2e7c31f04","spaceLabel":"Acme Engineering","identityKind":"user","destination":{"mode":"graph","topicIds":["topic-8f21"],"topicName":"traderevolution","gated":true,"enrichmentOf":"artifacts","modeChangedAt":"2026-08-05T09:11:40Z"},"planTier":"500MB","capabilities":{"noteAttachments":true,"noteBodyOnCreate":true,"probeNoteId":"note_4410"},"phase":7,"execution":"sequential","totals":{"rawFiles":4812,"items":1086,"bytes":6612345678},"consent":{"granted":true,"at":"2026-08-04T14:19:02Z","shownBytes":6612345678,"shownItems":1086},"enrichment":{"offered":true,"offeredAt":"2026-08-05T09:10:02Z","eligibleItems":874,"estimateShown":"3-5h","decision":"accepted","decidedAt":"2026-08-05T09:11:40Z","schemaConsent":{"granted":true,"at":"2026-08-05T09:28:11Z","shownAddedTypes":3,"shownNodeEstimate":4200}},"lastHeartbeat":"2026-08-04T15:02:11Z"}
```

- `graphId` records the pre-existing space this run writes to. A resume verifies it and never re-targets.
- `identityKind` records what `naumu_whoami` reported in Phase 0: `"user"` for an OAuth or session identity, `"bot"` for an API key. It decides two things the rest of the run depends on - a bot cannot pass `topicIds` to `naumu_create_note`, so it cannot file into a topic at all, and a bot session is rate-limited to 60 requests per minute per account. A resume re-reads it rather than trusting the recorded value; the connection may have changed underneath the run.
- `execution` is `"sequential"` or `"fanout"`. It is a different field from `destination.mode` and the two are never interchangeable: `execution` says how many sessions do the work, `destination.mode` says what kind of content lands.
- `destination` records the Phase 0 answer. `destination.mode` is `"graph"` (artifacts plus nodes) or `"artifacts"` (notes and media only, no nodes of any kind). `topicIds` and `topicName` are `null` for a space-wide import, which passes `sharedWithSpace: true` on every note instead. `gated` is true only when the named topic was verified `closed` AND your account was verified as a member of it - by `isMember` where the server returns that field, otherwise by the closed topic appearing in your own `naumu_list_topics` result at all. A `default` or `open` topic files content without gating and records `gated: false`.
- `destination.enrichmentOf` and `destination.modeChangedAt` are absent on every ordinary run and written only when an artifacts-only run is continued as a post-import graph enrichment pass. `enrichmentOf` records the mode the run closed in - always `"artifacts"` - at the moment `mode` flips to `"graph"`, and `modeChangedAt` records when. Their presence is how a resume knows it is inside an enrichment pass rather than a fresh graph run, which matters because the run's notes and uploads are already terminal and must never be re-created. `topicIds`, `topicName` and `gated` do not change in the flip; the destination was settled once, in Phase 0, and stands for the whole run.
- `enrichment` records the one post-import offer an artifacts-only run makes, and is absent until Phase 10 makes it. `offered` and `offeredAt` say it happened; `eligibleItems` and `estimateShown` record the count and duration range actually quoted, both derived from the run's own numbers; `decision` is `"accepted"` or `"declined"` with `decidedAt`. A recorded `decision` is final for the session, and `"declined"` is never re-asked. `schemaConsent` is written only on acceptance and records the enrichment pass's own Phase 4 gate - `granted`, `at`, and what was shown (`shownAddedTypes`, `shownNodeEstimate`). It is deliberately separate from the top-level `consent`, which covers the artifacts pass and its byte total and is never rewritten.
- `capabilities` comes from the Phase 0 probe and is re-probed on every resume. `probeNoteId` names the throwaway note the probe wrote; a resume appends to that note rather than creating a second one. A capability that was never probed is recorded as `unprobed`, not guessed.
- `capabilities.noteBodyOnCreate` records whether this server's `naumu_create_note` advertises a `markdown` input, so a note lands with its whole body in one call instead of a create followed by `naumu_note_append`. It is read off the tool schema, never off a version string, and it is re-read on every resume like the rest of the block.
- `phase` is the phase to **re-enter**, and nothing else: `0` through `10`, then the sentinel `"done"`, which is what marks a run finished. The moment phase N closes, set it to N+1 - never leave it at N as a record of what just ran. The difference matters most at Phase 6: a `phase` of `6` on a resume means the pre-seed has not happened yet, and re-entering at `6` with the run's own nodes already in the map would seed them back as `preseed` rows and poison the import-created set. In artifacts mode it never takes the values `4`, `5` or `9` - those phases do not run. An accepted enrichment offer is the one thing that moves `phase` backwards: it is set to `3` in the same write that flips `destination.mode` to `"graph"`, and from there the graph phases run normally.
- `consent` is what lets a resume continue without asking again. If the target space, the destination, or `corpusRoot` differs from what was consented to, the gate runs again.
- `lastHeartbeat` gets touched each chunk, so a human can tell a stalled run from a slow one.

## manifest.jsonl

Two record shapes share the file. Raw lines are written in Phase 1 and revised in Phase 2; logical-item lines are appended in Phase 2. Both are re-revised in Phase 3 by appending a new line for the same id.

```json
{"id":"9f2a41c7b0de","path":"export/28257/ticket.json","bytes":48213,"mime":"application/json","sha256":"3b1f8c0a...","sha256Sampled":false,"kind":"json","partOf":"L_4c81a09e","ts":"2026-08-04T14:14:03Z"}
{"id":"L_4c81a09e","logical":true,"bucket":"tickets","staged":"staging/tickets/28257.json","sources":["9f2a41c7b0de","a71c93f0d2b8"],"tier":"ticket","reason":"per-ticket export with comment array","ts":"2026-08-04T14:22:10Z"}
```

- A raw `id` is the first 12 hex of the sha256 of `path` relative to `corpusRoot`. It is stable across runs and independent of content, so a re-import of the same tree keys the same way.
- A logical id is derived the same way and prefixed so the two never collide: `L_` plus the first 12 hex of the sha256 of the item's source raw ids, sorted and joined with newlines. Same sources, same id, every rebuild - that determinism is what makes `staging/` disposable and the ledgers stable across a resume. The per-directory loose-media note gets an `M_` id over the sha256 of its relative directory, so it has a ledger subject of its own.
- `path` is always relative to `corpusRoot`. Never store absolute paths.
- `sha256` is the full content hash under 50MB. Above that, hash the first and last megabyte plus the byte length and set `sha256Sampled: true`.
- `partOf` names the logical item that consumed a raw file. A consumed raw file gets no tier of its own.
- `tier` is one of `graph-extract`, `note-only`, `attach-caption`, `attach-only`, `ticket`, `skip`, `defer`.
- `dupOf` carries the id of the first item with the same content hash, when the tier is `skip` for that reason.
- `supersededBy` carries the id of the item that won a cross-format fold, when the tier is `skip` with reason `duplicate-rendering`. It is deliberately not `dupOf`: `dupOf` means identical bytes, while a superseded rendering is different bytes saying the same thing.
- **Last line wins per id.** That is what makes transform and triage revisable without ever editing a line.

## status.jsonl

The item lifecycle, plus the durable artifact ids an item produced. Its subjects are logical items and the raw files that no logical item consumed.

```json
{"id":"L_4c81a09e","status":"in_progress","by":"coordinator","at":"2026-08-04T14:41:55Z","note":"chunk 1 of 3"}
{"id":"L_4c81a09e","status":"in_progress","by":"coordinator","at":"2026-08-04T14:43:02Z","artifacts":{"nodeIds":["n_8812","n_8813"],"noteId":"note_5521"}}
{"id":"L_4c81a09e","status":"done","by":"coordinator","at":"2026-08-04T14:44:20Z","counts":{"nodes":9,"edges":14,"uploads":2}}
```

- Values: `pending` is the **absence** of any line for that id. From there: `claimed` -> `in_progress` -> `done` | `failed` | `deferred` | `skipped`.
- Append a fresh line whenever an item gains a durable artifact. The fold takes the **last** status and the **union** of artifacts across the id's lines - that union is what recovery reads to find the note it already created or the nodes it already added.
- `failed` carries a `reason`; `deferred` carries a reason code such as `over-agent-read-cap`, `over-plan-cap` or `server-lacks-note-attachments`.

### The lifecycle rule that matters

**Write `in_progress` before the item's first write to the space. Write `done` after its last.**

The window between those two lines is the only place a crash can leave partial work, and the ledger is honest about it: an item found at `in_progress` on resume is *known* to be partially applied. Reverse the order and a crash after `done` but before the work finishes leaves an item silently half-imported, with nothing on disk saying so.

### Record a note id the moment it exists

Append a status line carrying `artifacts.noteId` IMMEDIATELY after every `naumu_create_note`, before anything else happens to that note - whether the body rode along in the create call as `markdown` or still has to be appended. A crash in the gap between creating a note and recording it leaves an untracked note in the space and a resume that cheerfully creates a second one. Record the audience the create response reports on the same line, so a later reader can tell a shared note from a stranded private one.

If an `in_progress` item turns up with no recorded `noteId`, do not create a second note on faith. **Do not reach for `naumu_search` either** - it indexes graph nodes only, notes are not in that index, and there is no list-notes tool, so it will report "nothing found" for a note that plainly exists and you will create the duplicate this rule exists to prevent.

Instead: ask `naumu_ask` for the note by its exact title and adopt it if the answer names one. If that comes back empty or ambiguous, mark the item `failed` with reason `orphaned-note`, record the title in the status line, and surface it in `report.md` so a human can find and reconcile it. **Never create a replacement note.** An orphan the report names is a five-minute cleanup; a silent duplicate is a permanent one.

### Recovering a partial item, per tier

Re-enter every `in_progress` item in **verify-then-continue** mode: read what exists, add only what is missing.

| Tier | Recovery |
|------|----------|
| `graph-extract` | **Reconcile the item's unresolved `intent` rows first** (see the sibling map below), then re-run its chunks. Once every intent is resolved the sibling map maps already-created labels back to the same ids and re-adds collapse to no-ops - but only then. Skip the reconciliation and an intent that was written, executed, and never resolved re-adds up to 25 nodes as fresh duplicates. Before re-adding an edge, list the node's connections and add only what is absent. |
| `note-only` | Read the note recorded in `artifacts.noteId` and append only the sections that are missing. Never create a second note for the same item. |
| `attach-caption`, `attach-only` | `uploads.jsonl` decides, per the stage table below. Never re-PUT bytes already bound. |
| `ticket` | In graph mode the ticket node is created and recorded first, the transcript note second. Read the note and append only comments after the last one present. |
| any | An item that fails re-entry twice is marked `failed` with the reason and reported. Do not loop on it. |

## sibling-map.jsonl (graph mode)

The dedup index, and the crash-safe record of every node write the run attempts. Key is the lowercased label joined to the type. Artifacts-mode runs create no nodes and never write this file.

Two row types share the file. An `intent` row is written **before** a `naumu_add_node` chunk call; a resolving row is written **after** the response, one per node the call created.

```json
{"t":"intent","chunk":"L_4c81a09e#2","item":"L_4c81a09e","keys":["acme routing service|Service","routing incident 2025-11-04|Incident"],"by":"coordinator","at":"2026-08-04T14:42:04Z"}
{"k":"acme routing service|Service","id":"n_8812","origin":"reused-existing","chunk":"L_4c81a09e#2","by":"coordinator","item":"L_4c81a09e","at":"2026-08-04T14:42:10Z"}
{"k":"routing incident 2025-11-04|Incident","id":"n_9140","origin":"created","chunk":"L_4c81a09e#2","by":"coordinator","item":"L_4c81a09e","at":"2026-08-04T14:42:10Z"}
{"k":"acme billing service|Service","id":"n_7204","origin":"preseed","by":"preseed","item":null,"at":"2026-08-04T14:33:07Z"}
```

- **First write wins.** A later resolving row with the same `k` and a different `id` is a collision, queued for the Phase 8 merge.
- `origin` is **required** on every resolving row and says how the id got into the map:
  - `created` - this run called `naumu_add_node` and the node did not exist before.
  - `preseed` - seeded in Phase 6 from `naumu_filter` over the space's pre-existing nodes.
  - `reused-existing` - found mid-run by search or dedup and adopted; it belonged to the team before this run started.
  - `reused-mine` - resolved back to an id this run had already created.
- **The import-created set is `origin == "created"` and nothing else.** Every rule that turns on "nodes this import made" reads that predicate: the Phase 9 reparent set, the near-miss sweep, the merge candidates. Rows with `preseed` or `reused-existing` are the team's work - enrich them, never duplicate them, never merge them away, and never hand them to `naumu_batch_reparent`, which deletes the parent edge they already have. Without the field, an id adopted past the 200-row pre-seed cap is indistinguishable from one this run created, and Phase 9 tears team nodes out of their hierarchy.
- **Intent rows are what make the add path crash-safe.** The dangerous window is between the `naumu_add_node` call and the resolving rows: a crash or an ambiguous timeout in there leaves up to 25 nodes in the space that nothing on disk records, `naumu_search` cannot yet see them (embeddings are written asynchronously), and Phase 8's collision merge is structurally blind to them. The intent row carries the chunk id (item id plus chunk number) and the chunk's candidate dedup keys, so the next session knows exactly what to look for. It is the same trick `uploads.jsonl` plays by writing `presigned` before the PUT.
- **Resume rule for intents:** any `intent` row with no resolving rows against its `chunk` is unfinished. Before re-adding anything for that chunk, run a `naumu_filter` or `naumu_search` reconciliation for those exact labels, adopt whatever is already there as resolving rows with the right `origin`, and only then re-add what is genuinely missing. An intent row whose resolving rows are all present is closed and needs nothing.
- In fan-out, workers append optimistically and re-fold the file's tail from a tracked byte offset before each chunk. There is no locking; residual races are what Phase 8 exists for.

## uploads.jsonl

One row per upload, keyed so the same bytes going to two destinations stay separate.

```json
{"key":"e3b0c44298fca8bd7d0dd0e5c8e6cbd4b1e1a3f0d92b5c7ae4f8106b2d3c9f77|note|note_5521","item":"L_4c81a09e","stage":"presigned","attachmentId":"att_9931","expiresAt":1754319120000,"bytes":1841203,"destKind":"note","destId":"note_5521","at":"2026-08-04T14:47:31Z"}
{"key":"e3b0c44298fca8bd7d0dd0e5c8e6cbd4b1e1a3f0d92b5c7ae4f8106b2d3c9f77|note|note_5521","item":"L_4c81a09e","stage":"put_ok","attachmentId":"att_9931","destKind":"note","destId":"note_5521","at":"2026-08-04T14:47:49Z"}
{"key":"e3b0c44298fca8bd7d0dd0e5c8e6cbd4b1e1a3f0d92b5c7ae4f8106b2d3c9f77|note|note_5521","item":"L_4c81a09e","stage":"bound","attachmentId":"att_9931","destKind":"note","destId":"note_5521","at":"2026-08-04T14:47:52Z"}
```

- `key` is the **full lowercase sha256** of the bytes, then the destination kind, then the destination id, joined with `|`. Nothing is truncated: the same file bound into two notes is two rows, and the same file re-offered to one note is one.
- `expiresAt` is the Unix-ms integer exactly as the presign returned it. Do not reformat it into a date string; the comparison a resume makes is arithmetic.
- The fold takes the furthest stage reached per key.
- A `failed` row carries the status code and message: `{"key":"...","stage":"failed","code":413,"message":"...","at":"..."}`.

### Write stage table

| Stage | Written when | Also record |
|-------|--------------|-------------|
| `presigned` | after the presign returns, BEFORE the PUT starts | `attachmentId`, `expiresAt`, `bytes`, `destKind`, `destId` |
| `put_ok` | after the PUT returns success | nothing extra |
| `bound` | after the note write or `naumu_post_message` returns | the binding call's result id where there is one |
| `failed` | any refusal | the HTTP status and a one-line reason |

Writing `presigned` first is the whole trick: a session killed mid-transfer still leaves the attachment id and its expiry on disk, so the next session knows whether to bind, re-upload, or start over.

### Resume stage table

| Stage on disk | What to do |
|---------------|-----------|
| `bound` | Skip entirely. The embed is already in the note. |
| `put_ok`, not expired | Bind only: write the sole-line embed. Do not re-upload. |
| `put_ok`, expired | Re-presign, re-PUT, re-bind. |
| `presigned`, no `put_ok`, not expired | PUT the bytes, then bind. |
| `presigned`, expired | Re-presign, then PUT and bind. |
| `failed` with 413 | Leave deferred. **Never retry** - the size ceiling will not move inside this run. |
| `failed`, other cause | One retry. If it fails again, mark the item `failed` and report it. |

Re-embedding an id that is already bound is harmless. Binding an id that has expired, or that belongs to a different note, fails the **whole** write with an error naming the bad ids - so bind an item's embeds in their own call rather than merging them into an unrelated note write.

## merges.jsonl (graph mode)

Phase 8's collision merge is the only destructive sequence the whole run performs, and it ends in `naumu_remove_node`. It gets a ledger for the same reason the uploads do: a crash halfway through leaves a node stripped of its content but still in the graph, and a resume that cannot tell which half already happened either re-merges blind or abandons the orphan.

One line per stage, per collision, keyed on the pair.

```json
{"key":"n_9140|n_9207","survivor":"n_9140","loser":"n_9207","k":"routing incident 2025-11-04|Incident","stage":"queued","at":"2026-08-04T15:41:02Z"}
{"key":"n_9140|n_9207","stage":"content-merged","at":"2026-08-04T15:41:09Z"}
{"key":"n_9140|n_9207","stage":"edges-repointed","edges":7,"at":"2026-08-04T15:41:18Z"}
{"key":"n_9140|n_9207","stage":"edges-removed","at":"2026-08-04T15:41:22Z"}
{"key":"n_9140|n_9207","stage":"node-removed","at":"2026-08-04T15:41:24Z"}
```

- `key` is `<survivor>|<loser>`, both node ids, survivor first. The `queued` line carries the full context; later lines only need the key and the stage.
- The survivor is decided once, at `queued`, and never revisited: a `preseed` or `reused-existing` id always survives a `created` one, and between two `created` ids the earlier sibling-map row wins. Write the choice down before touching anything.
- The fold takes the furthest stage reached per key, exactly like the uploads fold.
- A `failed` row carries the stage it failed at and the error: `{"key":"...","stage":"failed","at":"...","atStage":"edges-repointed","message":"..."}`.

### Write stage table

**Every stage row is written BEFORE the step it names, not after it.** A merge row records intent, not completion - the same trick `uploads.jsonl` plays by writing `presigned` before the PUT. Writing after the step instead leaves one hole nothing can recover from: a crash between `naumu_remove_node` returning and the row landing leaves a loser that is already gone with nothing on disk saying the pair was ever touched, and the next session either re-merges a node that no longer exists or reports a collision that was already settled.

| Stage | Written before |
|-------|----------------|
| `queued` | anything at all happens to this pair; the survivor is chosen first and recorded on this row |
| `content-merged` | the `naumu_update_node` that folds the loser's content and attributes into the survivor |
| `edges-repointed` | the `naumu_add_edge` batch that moves the loser's edges onto the survivor |
| `edges-removed` | the `naumu_remove_edges_bulk` on the loser's remaining edges |
| `node-removed` | the `naumu_remove_node` on the loser |

Record the edge count on `edges-repointed` once the batch returns, by appending a second row for that stage. The count is bookkeeping; the stage row itself still goes first.

### Resume stage table

This table is the authority on picking a half-done merge back up; `ingest.md`'s Phase 8 write table points here for the resume rules.

**A recorded stage proves intent, not completion.** Because rows are written before their step, the step they name may or may not have run. So a resume verifies each recorded stage's effect against the graph before skipping past it, and re-runs the step when the verification fails. Every step below is idempotent when it is redone this way.

| Stage on disk | What to do |
|---------------|-----------|
| `node-removed` | Verify the loser is actually gone: `naumu_get_node` on the loser id must fail or come back empty. **Only if that verification passes is the pair done - skip it entirely.** If the loser is still there, the removal never ran: remove it now and close. |
| `edges-removed` | Verify with `naumu_list_node_connections` on the loser that its edges are gone. Remove whatever is still hanging off it, then remove the loser node. |
| `edges-repointed` | List both nodes' connections. Repoint every edge the loser still carries that the survivor lacks - repointing an edge that already exists is a no-op; assuming it does not is how edges get lost. Then remove the loser's edges and the loser node. |
| `content-merged` | Verify the survivor carries the merged content: `naumu_get_node` on both and check the survivor's content and attributes for what the loser held. If any of it is absent, redo the content merge idempotently - read the survivor, concatenate only the parts that are missing, write once. `naumu_update_node` overwrites, so the check has to happen before the append, never after. Then carry on with the edges. |
| `queued` | Only the intent was recorded and nothing was written. Re-verify both nodes still exist - the other side may have merged them already - and run the merge from the top. |
| `failed` | Do not retry inside this run. Report the pair, the stage, and both ids in `report.md`, and leave the graph as it stands. |

A pair that reaches `queued` and no further is the one state a human has to be told about even on a clean finish, because the graph still holds both nodes.

## schema.json (graph mode)

Written once, at the Phase 5 gate. Mid-run schema evolution is the only thing that ever rewrites it - atomically, `schema.json.tmp` then rename, with a trace line recording the rewrite. A resume never rewrites it.

Store the `types[]` entries **exactly as `naumu_get_schema` returned them**. Do not flatten them into `"REL -> Target"` strings: the parent stays NESTED inside `connections` as a structured object, and Phase 7's edge-direction gate parses that object. A flattened copy is a copy the gate cannot read, and it also stops being round-trippable back into `naumu_update_schema`.

```json
{"frozenAt":"2026-08-04T14:31:00Z","graphId":"7c1e9b40-...","existingTypes":["Service","Person","Decision","Document"],"addedTypes":["Incident","RunbookDoc","ExternalVendor"],"types":[{"type":"Incident","description":"An unplanned interruption to a running service.","connections":{"parent":{"relation":"AFFECTED","target_node":"Service"},"required":[{"relation":"REPORTED_BY","target_node":"Person"}],"suggested":[{"relation":"FOLLOWS","target_node":"Incident"}]},"attributes":[{"name":"severity","type":"select","values":[{"label":"sev1"},{"label":"sev2"},{"label":"sev3"}]},{"name":"openedAt","type":"date","values":[]},{"name":"resolvedAt","type":"date","values":[]}]}]}
```

Three details the gate depends on: the type's name is on `type`, not `name`; a connection carries either `target_node` or `polymorphic: true`, never both and never a list; and a polymorphic connection means "any type", which the gate reads as `*`. A type with no `connections.parent` at all is a root, not a mistake to be repaired here.

`existingTypes` versus `addedTypes` is the line the run must not cross: a resume halts only when an **added** type went missing or changed, and tolerates new types the team created independently. The check is containment against what the file records *now*, mid-run evolution included - not against what Phase 5 first froze.

## trace.md

One line per event, hyphen separator:

```
[14:12] phase 0 preflight - space=Acme Engineering 7c1e9b40, dest=#traderevolution mode=graph, runDir=./.naumu-import/acme-export-20260804-1412, noteAttachments=yes, noteBodyOnCreate=yes, planTier=500MB
[14:22] phase 2 transform - 812 articles, 178 tickets, 96 docs staged from 4,812 raw files
[14:31] phase 5 gate - 12 types (9 pre-existing intact, 3 added), kind-audit clean, schema.json frozen
[14:43] phase 7 item L_4c81a09e chunk 2 - 6 added, 3 merged, 11 edges, 2 uploads bound
[15:58] phase 8 reconcile - 1,086 items terminal, 6 collisions merged, 1 orphan upload bound
[09:10] phase 10 enrichment-offer - 874 items eligible, estimate 3-5h, answer=accepted
[09:11] phase 3 enrichment-start - mode artifacts -> graph, enrichmentOf=artifacts, re-triage of 874 eligible items
```

The trace is the narrative channel; the ledgers are the machine channel; `todo.md` is the at-a-glance channel. None of them replaces the others, and chat replaces none of them.

## report.md

Written in Phase 10, for a teammate reading it months later: what landed, what was deferred and whether a plan upgrade would change that, what was skipped, what failed, pre-existing shape problems left deliberately alone, and the resume command.

An enrichment pass's own Phase 10 **appends** a second dated section rather than rewriting the file. The artifacts pass's account is what explains the notes, and it stays readable exactly as it was written.

## locks/

`locks/run.lock` exists on **every** run, sequential and fan-out alike. Phase 0 writes it the moment the run directory is created, carrying the writing session's identifier and a timestamp, and a session that finds a live `run.lock` over the same corpus that it does not own stops and says so rather than proceeding. Two runs over one corpus register schema additions against each other and trip each other's Phase 5 integrity check, and neither can see the other's ledger.

`locks/<shard>.claim` is fan-out only: a worker heartbeat, advisory - file locking is not portable across harnesses, which is why sharding is by locality and reconcile catches the rest. `fanout.md` owns the claim protocol and what a stale claim means.

## shards/ (fan-out only)

`shards/<id>.json` carries the static assignment - shard id, runId, graphId, corpusRoot, the sharding basis and directory roots, the logical item ids, and per-tier counts. `fanout.md` defines the exact shape and the sharding rules.

Assignment is static: a shard belongs to one worker for the whole run.

## Resume protocol

1. **Locate the run.** With `--resume <runDir>`, use it. Otherwise take the most recent directory whose `run.json.phase` is not `done`; if several are unfinished, ask which.
2. **Read `run.json`.** Verify the recorded `graphId` is reachable and the `corpusRoot` still exists. **Never re-target a different space** and never substitute a moved corpus - stop and ask. Re-verify a gated destination exactly as Phase 0 did (topic still exists, still closed, account still a member) and halt loudly if any of it changed; continuing a gated import into a topic that stopped gating is a leak, not a convenience. A `destination.enrichmentOf` of `"artifacts"` means this is an enrichment pass over a completed artifacts run: the notes and uploads are already terminal and are never re-created, and the offer in `run.json.enrichment` is already answered, so do not make it again.
3. **Rebuild `staging/` if it is gone**, by re-running Phase 2 over the same raw corpus. It is derived, so this is safe and produces the same items. If staging is present, leave it alone.
4. **Fold the ledgers.** Read each `.jsonl` line by line, discarding any trailing line that is not valid JSON. Manifest: last line wins per id. Status: last status plus the union of artifacts per id. Sibling map: first write wins, later differing ids are collisions, and every `intent` row without resolving rows goes on the reconciliation list before any re-add. Uploads: furthest stage per key. Merges: furthest stage per pair.
5. **Verify the schema** (graph mode). `naumu_get_schema` must still contain every type `schema.json` currently records, with its parent and connections intact. Never rewrite the file to make the check pass. Extra types are fine - the team may have added their own. A missing or changed **added** type halts the resume loudly.
6. **Re-probe capabilities** and overwrite `run.json.capabilities` - `noteAttachments` by re-running the four-step probe against `probeNoteId`, and `noteBodyOnCreate` by re-reading the `naumu_create_note` tool schema for its `markdown` field. Detect the field; never infer either one from a server version string. A downgrade defers the affected items and is recorded; it does not fail the resume.
7. **Check consent.** Recorded consent plus an unchanged space, destination and corpus root means continue without asking. Anything else means run the consent gate again.
8. **Report the counts** in one line: done, pending, partially applied, failed, deferred, skipped, plus the phase being re-entered and the space being written to.
9. **Re-enter at `run.json.phase`, partial items first**, in verify-then-continue mode, per the per-tier table above.
10. **Never redo the schema phases.** They ran once, the gate passed, the file is frozen.
