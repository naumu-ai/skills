# Fan-out

Read this only if the user accepted a parallel run at the **Phase 6 Plan the run**
gate. Sequential is the default and is correct for almost every import.

## When to offer it at all

All three have to be true:

1. The manifest holds more than roughly 400 logical work items (skipped and deferred
   lines do not count), or more than roughly 150MB of `graph-extract` text.
2. The harness actually exposes subagents. If it does not, do not mention fan-out;
   there is nothing to offer.
3. The user says yes explicitly, at the plan gate, after seeing the trade.

State the trade in one breath: parallel finishes sooner and produces some duplicate
entities that Phase 8 Reconcile has to merge afterwards. Sequential is slower and
cleaner. If the user is indifferent, run sequential.

Fan-out never starts implicitly, never mid-run, and never on resume without the
user saying so again.

**On an API-key (bot) session, fan-out buys throughput it cannot spend.** The rate
limit is 60 requests per minute **per account**, and every worker on that key
shares one budget - five workers do not get five budgets, they get one twelfth of a
second each. What fan-out still parallelizes is the local half of the work: reading
and parsing staged items, holding corpus bodies out of one context window, and the
per-item judgment that decides what to write. The writes themselves queue behind
the same ceiling they would sequentially. Say that plainly when you make the offer
on a bot session, and set the expectation as "less context pressure, roughly the
same wall clock", not "three times faster". An OAuth user session is not held to
that limit and does get real write parallelism.

## Who does what

| | Coordinator | Workers |
|---|---|---|
| Phases | 0 Preflight and destination, 1 Survey, 2 Transform, 3 Triage, 4 Schema fit, 5 Schema gate, 6 Plan the run, 8 Reconcile, 9 Restructure, 10 Close | 7 Ingest, for their shard only |
| Schema | owns it: reads it, extends it, freezes it to `schema.json` | never calls a schema tool, for any reason |
| Structure | owns all reparenting, edge removal, and node removal | never reparents, never removes a node or an edge |
| Pre-existing nodes | applies every enrichment, once, in Phase 8 | never calls `naumu_update_node` on one; requests it instead |
| Ledgers | folds all of them, owns `run.json` | append-only, tagged with their shard |
| Restructure | runs it once, after reconcile | disabled, including the per-item mini pass |

In artifacts mode there is no schema and no sibling map: workers create notes and
uploads only, and reconcile is the coverage fold plus the upload sweep.

Workers read `schema.json` from the run directory. It is frozen at the Phase 5
gate and it is the only schema they get. Mid-run schema evolution is a
sequential-only feature and is off in fan-out.

When a worker meets something the frozen schema has no type for, it does two
things and moves on:

- writes a trace line `[HH:MM] phase 7 schema-gap - <what> (by: <shard>)`
- routes the node to the nearest valid existing type rather than skipping it

The coordinator reads every `schema-gap` line during Phase 8 Reconcile, decides
whether the gap deserves a type, and adds it there. Workers never make that call.

## Sharding

**Shard by top-level directory first.** Locality is the point: entities repeat
inside a directory far more than across directories, so keeping a directory whole
inside one shard is what prevents most duplicates before reconcile ever has to
work. Assign whole top-level directories to shards, balancing on item count. A
logical item belongs to the directory its sources came from.

**Fall back to round-robin when the corpus is lopsided.** If any single top-level
directory holds more than 60% of the work items, directory sharding cannot balance
anything. Sort all items by path and deal them round-robin across the shards.
Path-sorted order keeps neighbours together, which recovers part of the locality
you gave up.

**Tickets shard together by key prefix.** All of `PROJ-*` goes to one shard. Ticket
links point at siblings inside the same project far more often than across
projects, and splitting a project across shards guarantees duplicate Person and
Issue nodes.

**Default 3 workers. Hard maximum 5.** More than that spends more on reconcile
than it saves on ingest, and it multiplies write pressure on one space.

**Assignment is static.** Write it once at the end of Phase 6 and never rebalance
mid-run. One file per shard, `shards/<id>.json`:

```json
{
  "shard": "s1",
  "runId": "acme-docs-20260804-0912",
  "graphId": "…",
  "corpusRoot": "/abs/path/to/corpus",
  "basis": "top-level-dir",
  "roots": ["docs/", "specs/"],
  "assignedAt": "2026-08-04T09:12:00Z",
  "items": ["L_4c81a09e", "L_0f9e8d7c", "…"],
  "counts": { "items": 137, "graphExtract": 88, "noteOnly": 21, "attachCaption": 24, "ticket": 4 }
}
```

`items` holds logical item ids, not paths, and the raw files those items own travel
with them. A worker works its list and nothing else - it never reads another shard's
file and never picks up an item it was not given, even if that item is obviously
stalled.

## Shard claims

`locks/run.lock` is not a fan-out feature and is not written here: Phase 0 writes
it when the run directory is created, on every run, and `state.md` owns it. By the
time fan-out is offered it already exists and already names this coordinator.

What fan-out adds is one claim file per shard, under the same `locks/`, plain text.

`locks/<shard>.claim` is a heartbeat. The worker writes it when it claims the
shard and refreshes the timestamp at each chunk boundary. A claim whose timestamp
is more than 15 minutes old means a dead worker. The coordinator does not silently
seize it: it reports the stale shard to the user, names how many of its items are
`done`, and offers to reassign the remainder. Reassignment rewrites that shard's
file and clears the claim.

## Sharing the sibling map without locks

Every worker appends to the same `sibling-map.jsonl`. There is no file locking -
it cannot be relied on across harnesses and operating systems, so the design does
not use it.

The protocol is optimistic:

- Each worker tracks the byte offset it has already folded.
- At the start of every chunk, it reads from that offset to end of file, folds the
  new lines into its in-memory map, and updates the offset. That is the tail
  refold, and it is cheap because it only ever reads new bytes.
- Each chunk writes its `intent` row before the add and its resolving rows after,
  exactly as `state.md` defines them, with `by` set to the shard id and `origin`
  on every resolving row. First write wins on a key.
- A key that arrives with a different id than the one already folded is a
  collision. The worker keeps using the id it already folded, and lets the
  collision stand in the file.

Two workers can still create the same entity in the same second, before either
sees the other's line. That is expected and it is not a bug to engineer around.
Phase 8 Reconcile folds the whole map, finds the collisions, and merges them.
Trying to prevent them with locking costs more than the merge does.

## Enriching a node the team already had

Phase 8 knows how to merge two nodes that were created twice. It has no answer for
one node that was *updated* twice, and `naumu_update_node` overwrites rather than
appends - so two workers enriching the same pre-existing node in the same minute
means the second one silently erases the first one's addition and the team's
original content along with it. There is no collision to detect afterwards and
nothing on disk saying it happened.

So workers do not enrich pre-existing nodes at all. When a worker matches a
sibling-map row whose `origin` is `preseed` or `reused-existing`, it appends an
**enrich-request** line to its own shard ledger, `shards/<id>.enrich.jsonl`, and
moves on:

```json
{"nodeId":"n_7204","k":"acme billing service|Service","origin":"preseed","item":"L_4c81a09e","by":"s2","add":"Handles retry-on-decline for EU card rails.","at":"2026-08-04T14:52:11Z"}
```

The coordinator reads every shard's enrich file during Phase 8 Reconcile, groups
the requests by `nodeId`, and applies each node's additions **once, merged** - one
read, one concatenation of everything every worker wanted to add, one write. That
is also the only place the read-then-append discipline can hold, because only the
coordinator is single-threaded enough to make the read and the write adjacent.

Nodes with `origin: "created"` are this run's own and workers enrich them freely;
the rule is about the team's nodes, which the run does not own.

`origin: "reused-mine"` resolves back to an id this run already created, so it is
run-owned too and takes the `created` rule: workers enrich it freely, no
enrich-request. Only `preseed` and `reused-existing` go through the coordinator.

## What workers write

Workers append to `status.jsonl`, `sibling-map.jsonl`, `uploads.jsonl`, and
`trace.md`, all append-only, every line tagged `by: <shard>` so the coordinator can
attribute anything it finds:

- `status.jsonl` - `claimed` when the shard starts an item, `in_progress` before
  the item's first write, then `done`, `failed`, `deferred`, or `skipped`.
- `sibling-map.jsonl` - as above, `intent` rows before each add and resolving rows
  after, each resolving row carrying its `origin`.
- `shards/<id>.enrich.jsonl` - enrich-requests against pre-existing nodes, for the
  coordinator to apply in Phase 8. The one shard file a worker writes rather than
  only reads.
- `uploads.jsonl` - the normal `presigned` / `put_ok` / `bound` / `failed` stages
  from `attachments.md`, keyed the same way.
- `trace.md` - the normal `[HH:MM] phase <n> <event> - <detail>` lines, with the
  shard tag in the detail.

Workers never touch `run.json`, `schema.json`, `manifest.jsonl`, `staging/`, or
another shard's files. `run.json` belongs to the coordinator alone, which is what
keeps its atomic write meaningful, and `staging/` is finished before any worker
starts.

## Restructure comes after reconcile

All restructuring happens once, in the coordinator's Phase 9, after Phase 8 has
merged collisions and swept near-miss labels. The order is the whole point:
reshaping duplicates you are about to merge away is wasted work.

The coordinator reports fan-out honestly in `report.md`: shards, worker count,
items per shard, collisions merged, `schema-gap` lines and what was done about
each, and any shard that went stale.
