# Restructure

Read this at the start of Phase 9, and in sequential mode before the first mini-restructure pass
in Phase 7. **Graph mode only** - an artifacts-only run creates no nodes and has no shape to fix.

Phase 9 fixes shape without changing meaning: reparent only, never delete a node, never rewrite
content. It runs once, on the coordinator, after Phase 8 has closed the run's writes.

## Scope (read this before anything else)

The space was not empty when you arrived. Restructure is **scoped to the nodes this import created
and to nodes this import made dense**. Concretely:

- You may reparent a node THIS RUN created.
- You may create mini-hubs and move this run's children under them, including when the dense parent
  is a pre-existing node - the parent gains a mini-hub, its old children stay where they are.
- You may NOT reparent a pre-existing child the import never touched, not even to make a cluster
  tidy, and not even when it clearly belongs under a mini-hub you just made.
- You may NOT remove a node, merge a pre-existing node, or alter an existing relation's direction.
- A shape problem the import did not cause is REPORTED, never acted on. Write it into `report.md`
  under pre-existing observations with node ids and one sentence of what you would have done. The
  team decides.

Derive the import-created id set from `sibling-map.jsonl`: **the rows whose `origin` is `created`,
and only those.** Every row records how its id got there, so there is nothing to subtract - `preseed`
and `reused-existing` are the team's nodes, `reused-mine` is already in the created set under its own
row, and an intent row that never resolved is not an id at all. `state.md` defines the line shape and
the `origin` values. If a row is missing an `origin`, or you cannot otherwise tell whether a node is
yours, treat it as pre-existing.

The space's existing root is audited only for the children THIS RUN added. If the run's additions
pushed the root past the thresholds below, group the run's additions into themed mini-hubs under the
root. Never re-shape the children the team put there.

## Phase 9a - Document grouping pass (runs first)

The `note-only` tier produces one Document-typed node per long-form item. A corpus with two hundred
long-form items therefore produces two hundred nodes that will flatten whatever they parent to,
before any other clause has a chance to fire. Do this pass before dense-node detection.

1. List the Document-typed nodes this run created.
2. Group them by source-derived theme: originating directory, staging bucket, subject area, document
   family (specs, meeting records, contracts, runbooks). Directory locality is usually the truest
   signal because it reflects how the team already filed the material.
3. Create one mini-hub per group, typed to the Document type, named for what the group actually is
   ("Q3 Board Records", "Onboarding Runbooks"), with one or two sentences of content.
4. Attach each mini-hub under the parent the documents currently share, with `naumu_add_edge`
   carrying `isParent: true`, then move the documents with `naumu_batch_reparent` in chunks of at
   most 25, folding the per-node response as in 9c step 5.

Trace: `[HH:MM] phase 9 doc-group - <N> Document nodes grouped into <M> mini-hubs`

## Phase 9b - Hub detection

### Step 0 - root audit (mandatory, runs first)

Audit the root BEFORE calling `naumu_list_dense_nodes`. The root is the one node whose flattening
costs the most and the one the dense-node scan is least likely to surface first, so it does not wait
its turn in a sorted list.

Count the children THIS RUN added directly to the root - created ids from `sibling-map.jsonl` whose
parent edge points at the root. That count has a hard cap of **twenty**. Over it:

- **KEEP is not available** for the run's own root children. Not "the types already segment it" -
  type-level segmentation is not grouping. Not "no clear theme". Group them.
- Build themed mini-hubs from facets the corpus actually supports (source families, partners,
  competitors, publications, incidents, releases, jurisdictions) and move the run's root children
  into them with the 9c procedure.
- Aim to leave the root with no more children than it had before the import, and never more than
  twenty added by this run.

Pre-existing root children are untouched, as everywhere else. If the root is over the cap only
because of what the team already put there, that is a `report.md` observation, not your work.

Trace: `[HH:MM] phase 9 root-audit - root <label>: <mine> children added by this run (<total> total), <verdict>`

The audit runs whether or not the root appears in the dense-node scan. Its absence from `trace.md`
fails Phase 9b exactly like a missing hub-audit.

### Dense-node detection

Call `naumu_list_dense_nodes(graphId, minConnections=10)` once. Sort by same-typed child count
descending, then by total children descending.

The tool counts CHILDREN, on parent edges only - mesh cross-links are invisible to it. Its
`connection_count` field is that total-children count despite the name; this file calls it **total
children** throughout, and `same_typed_child_count` the same-typed subset of it.

For each row, decide which clause applies:

```
if same_typed_child_count >= 10:    same-typed split
else if connection_count >= 20:     thematic split
else:                               not a hub, skip
```

- **Same-typed clause.** A node with ten or more children of the SAME type on parent edges should
  have those children split into same-typed mini-hubs grouped by semantic commonality.
- **Total-children clause.** A node with twenty or more total children of any mix of types should
  have them grouped into themed mini-hubs typed to each cluster's dominant child type.

Both clauses count ALL of a node's children, because that is what makes it hard to read - but only
this run's children may be MOVED. If a dense node's breach comes entirely from pre-existing children,
the verdict is KEEP with the reason `pre-existing, out of scope`, plus a line in `report.md`.

For each qualifying hub, build a plan: `naumu_list_node_connections(graphId, hub.id, direction:"in")`
to enumerate incoming edges; for a same-typed split filter to incoming parent edges whose other end
shares the hub's type; for a thematic split gather all incoming parent-edge children. Read the
content of three to five sample children per cluster with `naumu_get_node` before naming anything.

Trace: `[HH:MM] phase 9 hub-scan - <label> (<type>): <total> children (<mine> from this run), <same_typed> same-typed, plan <N> mini-hubs via <clause>`

### Mandatory hub-audit trace (no silent abstains)

For EVERY row with ten or more same-typed children or twenty or more total children, write an
audit BEFORE splitting it or advancing past 9b:

```
[HH:MM] phase 9 hub-audit - <label> (<type>): <same_typed> same-typed, <total> total, <mine> from this run
  thematic options considered: <option 1>, <option 2>, <option 3>
  verdict: SPLIT into <mini-hub list with counts> | KEEP (reason: <specific reason>)
```

**Acceptable KEEP reasons** - specific to this cluster, never boilerplate:

- "The source enumerates these as one entity class with no further hierarchy" (an alphabetical
  roster, a flat reviewer list).
- "Source-derived themes would each hold three or fewer members; sub-splitting would manufacture
  categories rather than reduce fragmentation."
- "Cluster is already at terminal depth; the children are leaves with no further accumulation
  expected."
- "The breach comes from pre-existing children this import did not touch" - out of scope, reported.

**Unacceptable**: silence, "abstained", or "no clear theme" with no list of the themes you weighed. A
soft cap means "audit the breach", not "ignore it". A KEEP without a concrete reason counts as a
missing audit and fails Phase 9b.

**Root exception.** The root was already settled in Step 0, under the hard cap and the KEEP ban
stated there. If it also surfaces in the dense-node scan, do not re-adjudicate it here - Step 0's
verdict stands.

## Phase 9c - Mini-hub creation and reparenting

For each cluster from 9b:

1. **Name it semantically**, from the cluster's own content: "Mobile App Features", "Direct
   Competitors", "iOS Bug Reports". Never generic - no "Feature Group 1", no "Other", no "Cluster A".
   A mini-hub whose name does not say what unifies the cluster is worse than no mini-hub.
2. **Type it** as the children's type (same-typed clause) or the cluster's dominant child type
   (total-children clause).
3. **Create it** with `naumu_add_node`, with one or two sentences of `content` describing the
   commonality.
4. **Parent it** under the original hub with `naumu_add_edge` carrying **`isParent: true`**, using the
   same parent relation the children used. Without that flag the mini-hub is a mesh cross-link
   hanging off the hub, not a place in the hierarchy - nothing is under it and the split changed
   nothing. Validate the direction against `schema.json` first; if that relation is not allowed at
   (mini-hub type -> hub type), use the schema-allowed equivalent and log the substitution.
5. **Move the children** with `naumu_batch_reparent(graphId, newParentId, newRelation, nodeIds)` in
   chunks of at most 25. The node list contains only ids from the import-created set.

   **Fold the response; it is per node, not atomic.** The call returns
   `[{nodeId, oldParentId, newParentId, status, error?}]` with `status` one of `moved`, `skipped`
   (already had that parent - idempotent, not a failure) or `error`. A call can come back with
   twenty moved and five errored, and the request tells you nothing about which. So:

   - Report `nodes_reparented` from the count of `moved` rows in the RESPONSE, never from the length
     of the request. A structural summary built from the request is fiction.
   - Retry `error` rows once. Anything still erroring is named in `report.md` with its `nodeId` and
     the returned `error`, and its children stay where they are.
   - Trace the fold: `[HH:MM] phase 9 reparent - <newParentId>: <moved> moved, <skipped> skipped, <errored> errored`

Trace: `[HH:MM] phase 9 hub-split - <hub label>/<clause>: <N> children grouped as <mini-hub label> via <relation>`
Abstained: `[HH:MM] phase 9 hub-skip - <hub label>/<clause>: <N> children, no clear commonality, left as-is`

**Recursion.** After a pass, if a mini-hub you created now has ten or more same-typed children or
twenty or more total children, re-run 9b and 9c on it. Cap at three levels; beyond that the taxonomy
is finer than the corpus supports.

**Edge mistakes are correctable.** If you find an edge with the wrong target, remove it with
`naumu_remove_edge` (or `naumu_remove_edges_bulk`) and add the correct one. Reach for these only on a
concrete mistake of your own, and only on edges this run created.

## Phase 9d - Island detection and collapse

An island is a set of this run's nodes that no path reaches from the space's root - typically a
cluster whose parent edges all got dropped by the direction gate.

1. Identify the root: the type in `schema.json` with no declared parent, and within it the node the
   space already treats as the root.
2. Walk UPWARD from the import-created set, not outward from the root. For each created node, follow
   parent edges with `naumu_list_node_connections` for at most five hops. A node that reaches an
   in-graph ancestor inside that budget is attached; a node that never does is sitting in a
   component. There is no full-space BFS here - a space with a hundred thousand pre-existing nodes
   must not be traversed to find out where this run's few hundred landed.
3. If the created set runs past roughly 500 nodes, sample it by type instead of walking every node,
   and say in `report.md` that the island sweep was partial.
4. For each component, sample members with `naumu_get_node` to learn its types and its theme.

Trace: `[HH:MM] phase 9 island-scan - component <N>: <count> nodes, types {<Type>: <count>, ...}, theme <name or unclear>`

To collapse a component:

1. Name a mini-hub from the component's content, under the same naming discipline as 9c.
2. Create it with `naumu_add_node` and attach it under the most appropriate in-graph parent - the
   root, or the pre-existing node the component is actually about - with `naumu_add_edge` carrying
   **`isParent: true`**. A mesh edge here would leave the mini-hub as much of an island as the
   component it was meant to rescue. Type it to the component's dominant type.
3. Identify the anchors: component members with no parent edge into any other in-graph node.
4. Reparent the anchors under the mini-hub with `naumu_batch_reparent`, folding the per-node response
   as in 9c step 5. Internal mesh edges are left alone; you are only adding parent edges.

Trace: `[HH:MM] phase 9 island-collapse - component <N>: <count> nodes anchored under <mini-hub label>`
Abstained: `[HH:MM] phase 9 island-skip - component <N>: unclear theme, left disconnected`

A component made entirely of pre-existing nodes is not yours. Report it, do not collapse it.

## Mini-restructure passes (sequential mode only)

In sequential mode, run a small version of this pass periodically during Phase 7. It keeps hubs from
compounding across an entire corpus.

**Cadence: every 25 items, or whenever more than 100 nodes have been added since the last pass,
whichever comes first** - plus one final pass at the last item of the run. Not once per item: on a
thousand-item corpus that is a thousand dense-node scans buying almost nothing between neighbouring
items, and it drowns `trace.md` in lines nobody reads.

Each pass:

1. `naumu_list_dense_nodes(graphId, minConnections=10)`.
2. If the root's child count crossed twenty because of this run's additions, run the Step 0 root audit
   from 9b now rather than waiting for the end of the run.
3. For every other row over the thresholds, run the 9b hub-audit and, on a SPLIT verdict, the 9c
   procedure - scoped, as always, to this run's children.
4. Emit the trace line for EVERY pass, even when nothing qualified:
   `[HH:MM] phase 7 mini-restructure - pass <N> (items <first>..<last>, <A> nodes added): <X> hubs evaluated, <Y> split, <Z> kept (audited)`
   A pass that fired with no line is itself a violation, exactly as a missing hub-audit is. The teeth
   are on the pass; an individual item owes nothing.

Verdicts recorded here are binding at Phase 9: a reasoned KEEP stands unless cross-pass evidence
appeared afterwards. Phase 9 then only handles accumulations no single pass could see.

**In fan-out mode this pass does not run** - see the worker constraints in `fanout.md`. Everything
lands in the coordinator's Phase 9 after Phase 8.

## Idempotence check

Running Phase 9 again on a finished run must be a no-op:

- `naumu_list_dense_nodes(minConnections=10)` returns no rows that qualify under either clause and
  are in scope.
- The 9d walk, run again, finds no components: every node this run created reaches an in-graph
  ancestor by following parent edges upward inside the five-hop budget. State it in that direction,
  the same one the procedure walks - there is no downward pass from the root to restate it against.
- `naumu_reparent` is idempotent: a child that already has the requested parent edge comes back as
  `skipped`, and a `naumu_batch_reparent` re-run returns all `skipped` and no `moved`.

## Structural summary

Append this line before Phase 10 closes the run:

```
[HH:MM] phase 9 done - <hubs_split> hubs_split, <mini_hubs_created> mini_hubs_created, <nodes_reparented> nodes_reparented, <islands_collapsed> islands_collapsed, <orphans_remaining> orphans_remaining, <pre_existing_reported> pre_existing_reported
```

`orphans_remaining` should be zero, and every node counted in `pre_existing_reported` is itemised in
`report.md`. Restate these numbers on the Phase 10 DONE line so the user can read the run's shape
outcome from one line.
