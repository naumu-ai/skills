# Ingest and reconcile

Read this at the start of Phase 7 and again before Phase 8.

This reference covers the chunk loop for `graph-extract` items and the reconcile pass that closes the
run's writes. Media tiers (`note-only`, `attach-caption`, `attach-only`) follow `attachments.md`;
`ticket` items follow `tickets.md`. All of them use the same ledger contract described here.

**The chunk loop is graph mode only.** In artifacts mode triage demoted every `graph-extract` item to
`note-only`, so Phase 7 is notes and uploads, and Phase 8 skips the collision and near-miss passes -
there are no nodes to collide.

## Preconditions

- `schema.json` is frozen (Phase 5). In fan-out mode the worker constraints in `fanout.md` apply to
  everything below.
- The sibling map is pre-seeded (Phase 6): for every type this run will write, `naumu_filter` pulled
  the space's existing nodes of that type and each one was appended to `sibling-map.jsonl` with
  `origin: "preseed"`. Without that seed, dedup only sees this run's own writes and the import
  duplicates content the team already has.

  Write the call out in full, per type - the defaults are not what this phase needs:

  ```
  naumu_filter({ graphId, nodeTypes: ["<Type>"], limit: 200, sortBy: "updatedAt" })
  ```

  `limit` defaults to 50 and `sortBy` to `sortKey`; 200 is the tool's CEILING, not its default, so
  omitting either argument seeds a quarter of the rows in an arbitrary order and the run's duplicate
  rate quietly quadruples. `naumu_filter` does not paginate, so a type larger than 200 is seeded from
  its 200 most recently updated; `run.json` records seeded-versus-total per type, and `report.md`
  names every truncated type as elevated duplicate risk.

  Guard the seed against itself: skip any returned id already present in `sibling-map.jsonl`. On a
  resume the map already holds this run's own creations, and re-seeding them would relabel nodes this
  import made as pre-existing and put them out of Phase 9's reach.
- The `edgeAllowed` lookup is built once, from `schema.json`, and cached for the whole run.

## Ledger contract

`state.md` owns the record shapes and the reasoning behind them. These are the rules this phase
applies:

- Write `in_progress` to `status.jsonl` BEFORE an item's first write, and `done` AFTER its last.
- Append `artifacts.noteId` the moment a note is created, before any content goes into it.
- Append exactly one trace line per chunk to `trace.md`.
- Append an intent row to `sibling-map.jsonl` BEFORE each `naumu_add_node` chunk call, and the
  resolving rows after it returns (see step 3). Both shapes are defined in `state.md`.
- Append every id you create or reuse to `sibling-map.jsonl`, each carrying the `origin` that says
  how it got there - `created`, `preseed`, `reused-existing`, `reused-mine` (see dedup, below).
  `origin` is required on every row: Phase 9 derives the import-created set from `origin == "created"`
  alone, and a missing or wrong value is what lets a reparent tear a team-owned node out of its
  hierarchy.
- On an unrecoverable item error, write `failed` with the reason and move to the next item. Do not
  halt the run for one item.

## The chunk loop

Work in chunks of at most 25 candidate nodes. Twenty-five is the bulk-tool ceiling, not a target;
smaller chunks give tighter per-batch verification.

### 1. Identify candidates

Extract entities and relationships from the current section of the staged item. Apply type discipline
as you go.

**Person discipline.** Every named human in the source becomes a node of the space's person-like
type, never a string inside another node's attributes. Quotes get a source; bylines get an author.

- A human attributed in a review or article ("Dr Smith wrote...", "by X for Y") gets a node AND an
  authorship edge to the piece, even when they appear only in aggregated content and never in a
  canonical author file.
- A first-name-only handle that appears three or more times as the speaker of human behaviour
  (replying, reviewing, supporting, posting) is a person, not an attribute string and not a concept.
- Parent edge versus employment edge are not redundant. The parent edge says where the person hangs
  in the hierarchy; an employment mesh edge says what the employment relationship is. Set both when
  the person has a known employer; set only the parent when they do not.

**Channel discipline.** Every distinct editorial source that publishes about the subject (review
site, news site, blog, podcast, video channel) becomes one node of the space's editorial-source type.
Aliases pointing at the same publisher - the name, the possessive form, a URL on that host - all map
to the SAME node, never one per mention.

**Coverage rule: type generality is not entity sparsity.** "Prefer general types" is a schema rule
and says nothing about how many entities to capture. If a section names 50 features, build 50
candidates; if it names 30 competitors, build 30. The source sets coverage, not a wish to keep the
graph small, and volume is handled with mini-hubs rather than by dropping entities. Logging
"long-tail enumeration intentionally dropped" is the wrong rule applied to the wrong thing.

**Hub prevention, in-chunk.**

- *Same-typed clause.* Before adding three or more same-typed children to a parent that already has
  seven or more same-typed children (the add would push it past ten), pause: is there a commonality
  worth a mini-hub? If yes, create the mini-hub in this same chunk with `naumu_add_node` (same type as
  the children, semantic name, one or two sentences of content describing the commonality), attach it
  under the original parent with `naumu_add_edge` carrying **`isParent: true`**, and route the new
  children to it - each routed child's parent edge to the mini-hub carries **`isParent: true`** as
  well. Without the flag the mini-hub is a mesh cross-link off the parent and nothing actually sits
  under it, so the split changed nothing; `restructure.md` spells out the same step.
  Trace: `[HH:MM] phase 7 hub-prevent - same-typed: 4 Features grouped under "Authentication Features"`
- *Total-children clause.* Before adding a chunk that would push a parent's TOTAL direct children
  past 20 (any mix of types), run a thematic-clustering pause. Group existing plus incoming children
  by source-derived theme, create themed mini-hubs typed to each cluster's dominant child type, attach
  each one under the original parent with `naumu_add_edge` carrying **`isParent: true`**, and reparent
  the clustered children into them so every child's parent edge to its mini-hub carries
  **`isParent: true`** too. This clause exists because a parent can accumulate a hundred mixed children
  while no single type ever crosses ten.
  Trace: `[HH:MM] phase 7 hub-prevent - total: <parent> at 22 direct children, split into 5 themed mini-hubs`
- *Abstain, but log the reasoning.* If no commonality or theme is clear, defer to Phase 9 - never
  silently. Trace: `[HH:MM] phase 7 hub-prevent - abstain: <parent> at <N> same-typed, considered
  themes [<list>], deferred to phase 9`. An abstain with no theme list reads as "did not try".
- Scope: mini-hubs you create in-chunk group children THIS RUN is adding. Do not sweep a
  pre-existing child into a new mini-hub here.

Hold the candidate list in mind; do not write yet.

### 2. Dedup pre-check (mandatory)

There is no server-side merge. `naumu_add_node` returns `status: "created"` for every input, always.
A previous server-side auto-merge was removed because it collapsed distinct entities that shared a
brand prefix in their labels, so dedup is entirely the importer's job.

For each candidate, in order:

**(a) Sibling-map lookup, first.** Key is `lowercase(trim(label)) + "|" + type`. The map is persisted
in `sibling-map.jsonl` and folded into memory at the start of the run and, in fan-out, refolded from
a tracked byte offset before each chunk. A hit means: drop the candidate from the add batch and reuse
the mapped id for this chunk's edges.

**A hit is only O(1) when the row is the run's own.** Read the row's `origin`:

- `created` or `reused-mine` - this run already resolved that key. Drop the candidate, reuse the id,
  no further judgment.
- `preseed` or `reused-existing` - the id belongs to a node the team owns, and an identical label is
  not proof of an identical entity. The team's `Project` "Atlas" and the corpus's vendor product
  "Atlas" normalise to the same key and would otherwise be fused silently, with the corpus's edges
  and prose landing on the team's node. Apply the same plausibility judgment step (b) demands:
  when the corpus context and the mapped node disagree about what the thing is, `naumu_get_node` and
  read its content before deciding. If they are not the same entity, create the corpus's node under a
  disambiguated label, record it with `origin: "created"`, and trace
  `[HH:MM] phase 7 dedup-split - <label> (<type>): map hit <id> is pre-existing and a different
  entity, created <newId>`.

The map is mandatory rather than optional because node embeddings are generated asynchronously after
a write. A node you added moments ago is not yet findable by meaning, so without the map two adjacent
chunks can each create the same entity.

**(b) `naumu_search` fallback, on a map miss.** Search the candidate label. Treat a result as the
same entity when ALL of the following hold:

- the result's `matchedVia` is `semantic` or `both` (a `text`-only match is a substring coincidence
  as often as not), AND
- the result's type is the candidate's intended type, AND
- the two labels plausibly name the same real-world entity, on your judgment, reading the result's
  content if the labels alone are ambiguous.

There is no numeric cutoff anywhere in this rule. The score returned by search is a rank-fusion
score, not a similarity, and comparing it against a threshold is meaningless. Judge the pair.

Two humans who share a first name are not the same person. A product and the company that ships it
are not the same entity. When genuinely unsure, add the node and let Phase 8's near-miss sweep flag
it rather than fusing two distinct things.

**(c) Enrich pre-existing matches, never duplicate them.** When the match is a node the space already
had (a Phase 6 pre-seed entry, or a search hit that is not in your created set), the corpus usually
knows something the node does not. Extend it with the new edges the corpus implies, and extend its
content and empty attributes with `naumu_update_node`.

**`naumu_update_node` OVERWRITES the fields you give it. There is no append mode.** Sending a
`content` of your new prose replaces everything the team wrote with it. The append is yours to
perform locally, and the read that makes it possible is mandatory, not a precaution:

1. `naumu_get_node` on the matched id. Record the pre-edit content and its hash.
2. Build the new content string locally: the existing content verbatim, then a delimited block

   ```
   ---
   Added by import <runId>
   <the corpus's contribution>
   ```

3. `naumu_update_node` carrying that COMBINED string, plus only the attributes that were empty.
   Never send an attribute that already holds a team value.
4. Write the pre-edit content hash onto the item's `status.jsonl` line. A resume compares the node's
   current content against that hash: matching means the append never landed and must be redone;
   differing, with the run's delimiter present, means it already did and must not be applied twice.

Trace: `[HH:MM] phase 7 enrich - <label> (<type>) existing node extended, <n> new edges, pre-edit <hash>`

**(d) Record the reuse.** Every dedup drop appends a line to `sibling-map.jsonl` carrying the reused
id under the candidate's key, so later chunks and other workers resolve the same label to the same
node. Record the `origin` that matches how the id was reached:

- `preseed` - the row Phase 6 already wrote for a node the space had. Leave it as it stands.
- `reused-existing` - a node the space already had, adopted through the step (b) search fallback
  (typically one that fell outside the 200-row pre-seed window). Without this value the id looks
  import-created and Phase 9 is free to reparent a team-owned node.
- `reused-mine` - a node THIS run created earlier, resolved again through search rather than the map.

First write wins; the same key later appearing with a different id is a collision and goes to the
Phase 8 reconcile queue.

Keep a chunk-local map of `droppedCandidate -> existingId` so step 4's edges route to the right nodes.

### 3. Bulk-add nodes, then update the sibling map

One `naumu_add_node` call, at most 25 nodes, carrying the candidates that survived step 2. Required
fields per node: `label`, `type`, and a non-empty `content`; `attributes` optional. Each call is
atomic - all 25 land or none do. If a chunk mixes concerns (fifteen people plus ten events under
different parents), prefer two smaller chunks.

**Every attribute you write must already be registered on its type.** `naumu_add_node` validates
attributes against the live schema, and ONE unknown key or one value outside a select attribute's
declared options rejects the entire batch with "no nodes were created" - the other twenty-four nodes
do not land either. Attribute registration is Phase 4's obligation (`schema.md` states it); if you
reach for a key that is not in `schema.json`, drop the attribute rather than the chunk, and note the
gap in `report.md`.

**Write the intent row first.** Append the intent row defined in `state.md` to `sibling-map.jsonl`
BEFORE making the call - the chunk id plus each candidate's dedup key. Only then call the tool. After
the response, append one resolving row per node with `origin: "created"`, carrying
`lowercase(trim(label)) + "|" + type -> id`, and update the in-memory map.

The window between those two writes is the only place in the run where up to 25 nodes can exist in
the space with nothing on disk saying so. The intent row closes it: on resume, an intent with no
resolving rows is reconciled per `state.md` - `naumu_filter` or `naumu_search` on those exact labels
FIRST, adopting whatever is already there, and only then re-adding what genuinely is not. Skipping
that reconcile duplicates the chunk, because node embeddings are asynchronous and the later collision
sweep cannot see what search cannot find.

**An ambiguous timeout is a crash, not a failure.** A call that times out may well have landed all 25
nodes. Never retry it directly. Treat it exactly like a resume: reconcile the intent row against the
graph, then add only the remainder.

### 4. Edge-direction validation, bulk-add, connectivity gate

**(a) The `edgeAllowed` lookup**, built once at Phase 7 start from `schema.json`.

Connections are STRUCTURED objects, not strings. Each type carries a `connections` object holding an
optional `parent` plus `required` and `suggested` arrays, and every entry in all three has the same
shape:

```
{ relation: "<RELATION>", target_node?: "<TargetType>", polymorphic?: true }
```

`target_node` and `polymorphic` are exclusive: a connection names ONE concrete target type or it is
polymorphic, never a list of targets. There is no flattened `"REL -> Type"` string anywhere in the
schema, so do not parse for one.

Fold all three into `edgeAllowed: (source_type, relation) -> [allowed target types]`:

- `connections.parent` - read `relation`, and `target_node` as the single allowed target.
- every entry in `connections.required` and `connections.suggested` - the same read.
- `polymorphic: true` means any target type is allowed; map that entry to `"*"`.

Keep the parent connection marked as such in the lookup. Step (c) needs to know which relation is the
type's parent relation, because that is the only relation an edge may carry `isParent: true` on.

**(b) Validate every candidate edge** before the bulk call. For `(source, relation, target)`:

- `edgeAllowed[(source_type, relation)]` missing or empty -> DROP the edge.
  Trace: `[HH:MM] phase 7 edge-gate - drop, no <relation> from <source_type>`
- target type not in the allowed list (and the list is not `["*"]`) -> check the reverse,
  `edgeAllowed[(target_type, relation)]`. If the source type is allowed there, FLIP the direction.
  Trace: `[HH:MM] phase 7 edge-gate - flip <source_type>/<target_type> via <relation>`
  Otherwise DROP.
  Trace: `[HH:MM] phase 7 edge-gate - drop <source_type> <relation> <target_type>, not in schema either direction`

**(c) Bulk-add the survivors.** One `naumu_add_edge` call, at most 25 edges, using ids from step 3
plus the reused ids from step 2.

**Every parent edge carries `isParent: true`. Every mesh edge omits it.** `naumu_add_edge` defaults
`isParent` to false, so an edge you meant as hierarchy but sent plain is a mesh cross-link: it does
not place the node under anything, `naumu_list_dense_nodes` will not count it, and `naumu_reparent`
will not manage it. Both kinds normally ride in the same call, so set the flag per edge:

- The edge that places a node under its parent - the relation `connections.parent` declares for that
  node's type - carries `isParent: true`.
- Everything else - an employment relation, an authorship relation, a blocks or relates-to link -
  omits `isParent` entirely. A mesh edge with the flag set would silently re-home the node.

**(d) Connectivity gate.** Every node created in step 3 must appear as the source or the target of at
least one edge once this step returns. Diff the step-3 ids against the edge endpoints. If the
difference is non-empty, add edges for each orphan immediately - a parent edge with `isParent: true`
where the schema defines a parent for its type, a mesh edge to a logical neighbour (no `isParent`)
otherwise - running them through (b) first.

**The remediation is bounded.** An edge batch is atomic, so a single bad edge fails all of them, and
retrying the same batch forever stalls the item:

1. First remedial attempt: the edges you judged best per orphan, as above.
2. Second and last attempt: fall back to the mechanical answer - parent every remaining orphan, with
   `isParent: true`, to the schema-declared parent target of its own type. Put each edge in its own
   small batch so one rejection cannot take the rest down with it.
3. Still orphaned after that: write `failed` to `status.jsonl` for the item with reason
   `orphans-unattachable` and the orphan node ids listed, trace
   `[HH:MM] phase 7 edge-gate - orphans unattachable on <itemId>: <ids>`, and move to the next item.
   The nodes stay in the space and Phase 9d's island sweep gets another pass at them.

Do not start the next chunk while orphans remain unresolved and attempts are still available.

Then take the next chunk of the same item. When the item's chunks are done, run the item close-out.

## `note-only` items - the note, then its Document node

A `note-only` item is not finished when its note is written. In **graph mode** every one of them also
gets exactly ONE node standing for it in the graph, so the corpus's long-form material is reachable
by graph navigation and not only by full-text search. Phase 4 was required to find that class a type
and a home; this is where both get used. (In artifacts mode there is no node - the note is the whole
landing - so skip everything below.)

Per item, in this order:

1. `naumu_create_note` with the destination's `topicIds` or `sharedWithSpace: true` AND `markdown`
   carrying the staged file's content - one call, note and body together. Append `artifacts.noteId`
   to `status.jsonl` immediately, before anything else touches that note. Embeds still come after,
   because a presign is made against a `noteId` that has to exist first (`attachments.md`).
2. Create ONE node of the Document type Phase 4 chose, through the same dedup pre-check and the same
   intent-row protocol as step 3 of the chunk loop. Its label is the document's own title, its
   `content` is a short description of what the document is, and the note's id rides as an attribute
   (whatever `schema.json` registered for it - `source_note`, `transcript_note`, the name the space
   already uses). That attribute is the only link from the graph back to the body, so it is not
   optional.
3. Parent it with `naumu_add_edge` carrying `isParent: true`, to the home Phase 4 chose for the
   `note-only` class. **Not to a mini-hub** - mini-hubs for these do not exist yet. Phase 9a groups
   them by source theme afterwards, and it can only do that once they all exist under a real parent.
4. Record BOTH artifacts on the same status line for the item - `artifacts.noteId` and
   `artifacts.nodeIds` - so a resume can tell a note with no node from a note with one.

Trace: `[HH:MM] phase 7 note-only - <itemId>: note <noteId>, Document node <nodeId> under <parent label>`

**The body is a blob, not a draft.** What goes into `markdown` is the staged file read from
`staging/` and passed through untouched - the same bytes, in the same order, whitespace and all. The
run never retypes a document through its own output to get it into a note. Copy-heavy corpora spend
most of their budget exactly there, and handing the file over verbatim in the create call is how you
stop spending it. Summarising, tidying, or re-rendering staged text is a defect, not a courtesy.

**Detect the field, not the version.** Read the `naumu_create_note` schema your harness actually
shows you. If it advertises a `markdown` input, take the single-call path above for every note this
run makes. If it does not, fall back to the older sequence - create the note empty, record
`artifacts.noteId`, then fill it with `naumu_note_append` calls - which is still correct, just more
expensive on both calls and tokens. Settle this once in Phase 0 as `capabilities.noteBodyOnCreate`
and read it from there; do not re-derive it per item and never gate it on a version string.

Where the document's body also names entities worth their own nodes, run the chunk loop over it as
well and edge them to this Document node. That is additive; the Document node itself is mandatory
either way. A graph-mode run that lands two hundred notes and zero nodes has not imported the corpus
into the graph, however cleanly it reports.

## Mid-run schema evolution (sequential mode only)

Phase 5 freezes the schema, but ingest sometimes reveals that a `kind` enum you ADDED has values with
genuinely different natural homes, or that you are about to create a node typed as a banned umbrella
name. In **sequential mode only**, you may evolve the schema in flight:

1. Trace the trigger: `[HH:MM] phase 7 schema-evolve - <Type>.kind values [<list>] diverged into
   <new types>, reason: <one sentence>`
2. Add the new types with `naumu_add_node_type`, declare parents and connections with
   `naumu_add_connection` - passing `kind: "parent"` explicitly on the parent connection, since the
   tool defaults `kind` to `"suggested"` and a type whose parent was registered as a suggestion has
   no declared parent at all. Optionally drop the orphaned values from the original enum with
   `naumu_update_schema`; existing nodes keep their old value as frozen artifacts of the catch-all.
3. Reparent nodes THIS RUN created whose kind moved, via `naumu_reparent` or `naumu_batch_reparent`.
   Never reparent a pre-existing node here.
4. Rewrite `schema.json` atomically - write `schema.json.tmp`, then rename it over the original -
   confirm with `[HH:MM] phase 7 schema-evolve - done, added [<types>], reparented <count>`, and
   continue. This is the only thing in the run that rewrites that file. A resume checks it as it
   now stands and never rewrites it.

Evolve when an added enum already has ten or more instances split across three or more semantic
homes and chunks keep adding more, or when a candidate obviously wants a specific type that does not
exist yet. Do not evolve for two or three nodes, and do not evolve from low confidence in the first
chunks of the first item.

**In fan-out mode this is disabled**, along with the rest of the worker constraints in `fanout.md`.
A worker that hits a schema gap logs
`[HH:MM] phase 7 schema-gap - <item>: <candidate> has no valid type, routed to <NearestType>`,
routes the node to the nearest valid existing type, and continues; the coordinator decides at
Phase 8 whether the gap deserves a type.

## Copy-through at scale

When many `note-only` items carry staged content that needs no per-item judgment - the text was
already normalized in Phase 2 and the note is a faithful copy of it - authoring every append by hand
spends the run's budget on transport, not thinking. Script the copy instead:

- The upload PUT already works this way: a shell command using the credentials the MCP server itself
  holds. A locally-run stdio server carries its API base URL and key in the configuration the harness
  started it with; when you can see those, a run-directory script can make the same note-write calls
  the tools make. When you cannot reach the API directly - a remote or OAuth-only server - batch the
  MCP calls and accept the cost.
- The script sends the body in the create call, exactly as the tool path does: where
  `capabilities.noteBodyOnCreate` is true it reads the staged file off disk and posts it as
  `markdown` alongside the audience, one request per note, and the content never passes through the
  conversation at all. Where it is false the script creates and then appends, same as the tool path.
- Consent comes first, always: the script runs only after the gate, and every note it creates carries
  the destination's `topicIds` or `sharedWithSpace: true`, exactly like a tool call would.
- The ledger contract does not bend: the script, or you immediately around it, writes the same
  `status.jsonl` lines - `in_progress` before an item's first write, `artifacts.noteId` the moment a
  note exists, `done` after its last - plus one trace line per item. A scripted write that skips the
  ledger breaks resume.
- Verify like transform: `naumu_note_read` a sample per bucket and compare against staging, and
  spot-check the counts. A script error found at Phase 8 costs a manual sweep; one found on the first
  sample costs a fix and a re-run.
- What never rides the script: alt text for `attach-caption` images (each needs a local read), entity
  extraction, dedup verdicts, and anything the staged content did not already settle.

Ticket transcript notes qualify - the comment transcript is staged, so copy-through applies - but
their per-comment attachment embeds still follow `attachments.md`.

## Item close-out

Before writing `done` for an item:

1. **Coverage.** Re-scan the staged item's section headers and bullet lists. Every enumerable entity
   named there - people, organisations, features, events, publications - must have a node. Abstract
   synthesis is your call; enumerated entities are not. If something was skipped, run one more chunk.
2. **Orphan sweep.** Confirm every node added for this item has at least one edge. This catches
   anything the per-chunk gate let through, for example an edge the server rejected after the chunk
   closed. Add edges before advancing.
3. In sequential mode, check whether this item closes a mini-restructure pass. The pass described in
   `restructure.md` runs **every 25 items, or whenever more than 100 nodes have been added since the
   last pass, whichever comes first** - and once more at the last item of the run, so nothing is left
   unaudited. It is not per item: a 1,086-item corpus would otherwise spend 1,086 dense-node scans on
   shape that barely moved between items. The mandatory trace line belongs to the PASS, not to the
   item. In fan-out mode, skip it entirely; all shape work happens in the coordinator's Phase 9.
4. Write `done` to `status.jsonl` and trace:
   `[HH:MM] phase 7 item-done - <itemId>: <added> added, <merged> merged, <edges> edges, orphans 0`

## Phase 8 - Reconcile

Runs once, on the coordinator, after every item is `done`, `failed`, `deferred` or `skipped`.

**1. Coverage fold.** Fold `manifest.jsonl` and `status.jsonl` into counts per status and per tier.
Every item must have a terminal status; an id still at `claimed` or `in_progress` means a worker
died - re-run that item now, or mark it `failed` with the reason. Trace:
`[HH:MM] phase 8 coverage - <done>/<total> done, <failed> failed, <deferred> deferred, <skipped> skipped`

**2. Sibling-map collision merges.** Fold `sibling-map.jsonl`. Any key holding more than one distinct
id is a race: two chunks or two workers created the same entity. **The survivor is decided by origin,
not by timestamp**: a `preseed` or `reused-existing` id always survives a `created` one, and only
between two `created` ids does the earlier sibling-map row win. The removal rule below and
`state.md`'s merge ledger both spell out the same order; never pick the survivor by age alone. For
each loser:

**This is the only destructive sequence in the run, so it is staged on disk.** Each merge gets rows
in `merges.jsonl`, whose record shape `state.md` owns. Write the stage BEFORE performing the step it
names, exactly like the uploads ledger writes `presigned` before the PUT:

| Stage | Written before |
|-------|----------------|
| `queued` | anything happens to this pair |
| `content-merged` | the `naumu_update_node` that folds the loser's content in |
| `edges-repointed` | the `naumu_add_edge` batch that moves the loser's edges |
| `edges-removed` | the `naumu_remove_edges_bulk` on what is left |
| `node-removed` | the `naumu_remove_node` |

The steps themselves:

1. `naumu_get_node` on both, to see what each holds.
2. Merge the loser's content and attributes into the canonical node with `naumu_update_node`. It
   overwrites, so concatenate locally first, exactly as the enrich step does.
3. Re-point the loser's edges onto the canonical node with `naumu_add_edge`, every edge going through
   the direction gate from step 4b first, and each one keeping the `isParent` value it had on the
   loser - a parent edge stays a parent edge.
4. `naumu_remove_edges_bulk` on the loser's remaining edges.
5. `naumu_remove_node` on the loser.

**A resume picks a half-done merge up at its recorded stage; it never restarts one.** Restarting
re-folds content that already landed, and a crash between `edges-removed` and `node-removed` leaves a
content-stripped orphan that a blind re-run would merge into the canonical node a second time. Read
the last stage for the pair, verify that stage's effect against the graph, and continue from there.
Because a row is written before its step, a recorded stage proves intent rather than completion -
**`state.md`'s resume stage table is the authority on what to verify at each stage**, and it is what a
resume follows.

Trace: `[HH:MM] phase 8 collision-merge - <key>: merged <loserId> into <canonicalId>, <n> edges re-pointed`

**Removal is only ever allowed for a duplicate node this import created** - a row whose `origin` is
`created`, and nothing else. If either side of a collision is a pre-existing node (`preseed` or
`reused-existing`), the pre-existing one is canonical by definition and the import-made one is the
loser. If BOTH sides are pre-existing, this is not your collision: leave both alone and note it in
`report.md`.

**3. Near-miss sweep.** Compare labels normalised (lowercased, trimmed, punctuation and common
suffixes stripped) across two sets, per type this run touched:

- **The run's own creations come from `sibling-map.jsonl`**, not from `naumu_filter`: every row with
  `origin == "created"`. That set is complete by construction, however many thousands of nodes the
  run made. Sweeping it through `naumu_filter` instead would cover 200 of them and silently call the
  rest clean.
- **The pre-existing side comes from `naumu_filter`**, which is the only way to see nodes this run
  never touched. Its 200-row ceiling applies here as it did at pre-seed, so a larger type is compared
  against its 200 most recently updated (`limit: 200`, `sortBy: "updatedAt"`) and named in
  `report.md` as elevated duplicate risk. Rows already in the sibling map need no second look.

Compare the created set against itself and against the pre-existing side. Pairs that normalise to the
same string, or that differ only by a legal suffix or an obvious spelling variant, are candidates.
Merge the confident ones with the procedure above. Report the ambiguous ones in `report.md` with both
ids and leave the graph untouched - a wrong merge is much more expensive than a duplicate. Trace:
`[HH:MM] phase 8 near-miss - <Type>: <n> pairs examined, <m> merged, <k> reported ambiguous`

**4. Orphaned-upload sweep.** Fold `uploads.jsonl` and resolve every entry that never reached its
terminal stage, per the resume table in `state.md`. Anything still unresolved is listed in
`report.md` with its reason. Trace:
`[HH:MM] phase 8 upload-sweep - <bound> bound, <retried> retried, <unresolved> unresolved`

Close the phase by regenerating `todo.md` from the folded ledgers.
