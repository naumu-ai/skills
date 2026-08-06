# Schema fit and the schema gate

Read this before proposing any schema change (Phase 4), and again before passing the gate (Phase 5).

**Both phases are graph mode only.** An artifacts-only run never reads the schema for fitting and
never writes to it; it goes straight from triage to the consent gate.

The target space already exists. Someone designed its schema for their own work, and the team's
nodes already hang under it. Your job is to make the corpus fit that schema, and to add only what
genuinely has no home. You are extending a running system, not designing a graph.

## Non-negotiables

- Never rename, remove, or repurpose an existing type. Not to tidy it up, not to make your mapping
  neater, not because you would have named it differently.
- Never change an existing type's declared parent, and never flip an existing connection's direction.
- Never re-root the space. The existing root stands. Do NOT introduce an abstract `Subject` container
  above it. That pattern belongs to a graph built from scratch; imposing it on a live space breaks
  every path the team already navigates.
- Never widen an existing type into a catch-all so it can absorb corpus entities. If the fit needs a
  stretch that big, it is an addition, not a fit.
- Do not model the corpus after a shape you have seen in some other space. A schema that worked
  elsewhere carries that space's compromises. Design against this corpus and this space's existing
  types, nothing else.

If you want to reshape what the team already built, keep the note and put it in `report.md` under
pre-existing observations at Phase 10. Reporting is allowed. Acting is not.

## Phase 4 - Schema fit

### 1. Read the target schema first

Call `naumu_get_schema(graphId)` before you look at a single corpus entity. Record, for every type:
name, declared parent connection, other connections, attributes, and any `kind` enum with its values.
This record is the baseline the gate checks and the base of `schema.json`.

Record it in the shape the tool actually returns, unflattened. Each entry under `types` carries a
`connections` object holding an optional `parent` plus `required` and `suggested` arrays, and every
connection in all three is `{ relation, target_node? , polymorphic? }` - a structured object, never a
`"RELATION -> Target"` string. Keep that shape in `schema.json`: Phase 7 builds its edge-direction
lookup by reading those fields, and flattening them here is what breaks it on the first chunk.

Trace: `[HH:MM] phase 4 target-schema - <N> existing types, root=<RootType>`

### 2. Inventory what the corpus will produce

From the manifest plus a high-level skim of a few staged items per bucket, list the entity CLASSES
the corpus contains: named humans, companies, products, features, incidents, releases, tickets,
publications, and so on. Classes, not instances. You are not extracting entities yet.

The `note-only` tier is itself one of those classes. Every long-form item it produces needs a
document-like node to stand for it in the graph, so list that class alongside the rest and give it a
verdict like any other: FIT it onto the type the space already uses for documents, or ADD one. A
graph-mode run holding `note-only` items must not leave Phase 4 without a home for them.

### 3. Map each class onto the existing schema

Every class gets exactly one verdict:

- **FIT** - an existing type already means this. Use it. Different vocabulary is not a different
  type: the corpus's "engineer", "reviewer" and "author" all map onto the space's existing
  person-like type.
- **STRETCH** - an existing type means this once you add an attribute, a `kind` value, or a
  connection. Additive only. Prefer a stretch over a new type.
- **ADD** - nothing existing covers it, and forcing it into a neighbouring type would misrepresent
  it. Design a new type under the discipline below.

Bias hard toward FIT. Every added type is a type the team has to live with.

Trace one line per class:
`[HH:MM] phase 4 map - <Class> -> FIT <Type> | STRETCH <Type> (+<what>) | ADD <NewType>`

### 4. Discipline for ADDED types

The rules below apply to types you add. They do not apply retroactively to what already exists.

**Banned umbrella names.** `Topic`, `Concept`, `Note`, `Item`, `Thing`, `Entry`, `Element`. These
names almost always mean a catch-all was about to be designed. If your sketch contains one, stop and
decompose. The same test applies to any name: write one short sentence describing what the type is.
If the sentence needs an "or", it is an umbrella and must be split.

**No umbrella differentiated only by a `kind` attribute.** Do not design a `<Type>.kind` enum whose
values span fundamentally different semantic categories. If the values name disjoint kinds of facts
(a praise signal, a strategic risk, a cross-cutting motif, a marketing stance), each category is its
own type.

An acceptable `kind` enum has values that are facets of one coherent dimension sharing one natural
connection shape, for example a publishing-surface type with `kind = {review_site, app_store, social,
podcast, blog}`, or a person-like type with `kind = {founder, executive, employee, contractor}`.

**Catch-all decomposition.** When sketching a `kind` enum, ask whether every value would attach to
the same parent. If some values would naturally hang off different parents, those are separate types.
An enum with five or more values that each want a different parent is a catch-all - split it before
registering anything. Polymorphic or any-type connections are a last resort for specific structural
purposes, never a default.

**Person-like types.** If the space already has one, use it as-is, even where its declared parent
does not suit every corpus human; route them to a parent the existing schema allows and note the
strain in `report.md`. If you are ADDING a person-like type, declare its parent polymorphic. Humans
appear in structurally distinct roles: employees of an organisation, authors or operators of a
publishing surface, and external contributors with no canonical employer. Pinning all three under an
employment relation forces you to misrepresent the last two. Declare the parent as a single
semantically neutral relation marked polymorphic, and keep employment as a separate mesh connection:

```
Person:
  connections:
    parent:    { relation: "ASSOCIATED_WITH", polymorphic: true }
    suggested:
      - { relation: "WORKS_AT", target_node: "Company" }   # mesh, only when employment is canonical
      - { relation: "AUTHORED", target_node: "Content" }
```

**A connection is a strict XOR: ONE concrete `target_node`, or `polymorphic: true`. Never a list.**
There is no `(Company | Channel | Root)` alternation - the API rejects it, and writing one is the
usual way a polymorphic parent ends up registered as nothing at all. When a connection is
polymorphic, the allowed targets are implicit: any type may sit at the other end, which is exactly
why the escape hatch is reserved for genuinely cross-cutting relations like this one and is never
the default. If you cannot pick a concrete target for a relation, you may be missing a type; add
that type first.

The Phase 7 edge-direction gate reads `polymorphic: true` as its `"*"` wildcard.

**Where additions hang.** Every added type needs a declared parent that points at a type in the
merged schema, existing or added. Prefer parenting into the existing hierarchy so the corpus lands
inside the space's structure rather than beside it.

**Reserved-sounding attribute names.** Never design an attribute named `visibility`,
`sharedWithSpace`, or anything else that reads like an access property - the server treats some of
these as access-control fields on the node itself, and a corpus value flowing into one can silently
widen or restrict who sees the node. Name the corpus's own concept something concrete
(`ticket_status`, `audience_label`) instead.

### 5. Every attribute Phase 7 will write must be declared here

Phase 4 owes the run more than types. **Every attribute name any tier will write in Phase 7 has to
exist on its type in `schema.json` before the gate closes** - added with `naumu_add_attribute` on an
existing type, or declared inline on a type you add.

This is not tidiness. `naumu_add_node` validates attributes against the live schema, and ONE
unregistered key, or one value outside a select attribute's declared options, rejects the WHOLE call
with "no nodes were created". At a chunk size of 25 that is 24 innocent nodes lost to one typo, and
the same failure repeats on every chunk until the schema is fixed - which in fan-out mode a worker
is not allowed to do.

So walk the tiers and list what they write, before the gate:

- `ticket` items write status, priority, created date, resolved date, reporter, assignee, source URL,
  and after the transcript `comment_count` and `transcript_note` (`tickets.md` prescribes the set).
- `note-only` items write the note id onto their Document node, under whatever name the space uses.
- `graph-extract` items write whatever the corpus's own metadata gives them - source URLs, versions,
  external keys, role decorations kept as attributes.

Two shapes to get right at declaration time, because they are the ones that fail at Phase 7:

- **Dates** are declared with type `"date"` and written as `"YYYY-MM-DD"` strings. A date attribute
  declared as free text will accept anything and then sort as text forever.
- **Select attributes** must declare every value the corpus actually contains. A ticket export with
  a status your enum does not list fails its whole batch; either declare the value or normalize it
  locally to one you did.

Trace one line: `[HH:MM] phase 4 attributes - <N> attributes declared across <M> types for phase 7`

## Phase 5 - Schema gate (hard)

### Register the additions

The consent gate closes Phase 4. Do not run any call below until the human has said yes.

Schema is registered with schema tools, never with data-write tools:

- `naumu_add_node_type`, then `naumu_add_connection` and `naumu_add_attribute` - incremental, one
  piece at a time. This is the safer default against a populated space: each call is additive and
  cannot disturb a type you did not name.

  **Declare the parent inline on `naumu_add_node_type` where you can.** If you declare it separately,
  `naumu_add_connection` needs `kind: "parent"` passed explicitly - the parameter defaults to
  `"suggested"`, so the obvious call registers the type's parent as an optional suggestion and leaves
  the type with no declared parent at all. That passes a careless reading of the gate and fails
  condition 3.
- `naumu_update_schema` - a single call carrying a full schema definition. Usable, but it must
  restate every pre-existing type exactly as `naumu_get_schema` returned it. If you are not
  confident you can echo the existing schema back unchanged, use the incremental tools.

  **The round trip is not literal; two keys are renamed and one is dropped.** `naumu_get_schema`
  returns `{ description, types: [...], space_description }`. `naumu_update_schema` takes
  `{ graphId, schema: { description?, nodes: [...] } }`. So: the read's `types` array goes in as
  `nodes`, it is wrapped in a `schema` object, and `space_description` is dropped entirely - it is
  folded in from the space summary for reading and is not part of the definition. Each entry inside
  the array needs no reshaping; per-type shape is genuinely identical in both directions.

  One thing this softens: **omitting a type's `connections.parent` PRESERVES its existing parent**
  rather than clearing it, so an accidental omission does not silently orphan a type. Sending
  `parent: null` explicitly is what removes one. The warning still stands for everything else -
  anything else you omit IS removed, and a type left out of `nodes` is deleted along with its
  connections.

`naumu_add_node` does NOT register a type. It creates an instance and copies the type string onto it
as a property. A type that only ever appears on instances stays absent from `naumu_get_schema` and
renders nowhere. Do not call `naumu_add_node` in Phase 4; instance data starts in Phase 7.

Trace: `[HH:MM] phase 5 schema-add - added <Type> (parent <RELATION> -> <Target>)`

### Verify the registration

This is a hard gate, not advisory. Call `naumu_get_schema(graphId)` again and check the response.

**Mandatory pass conditions.** All must hold before Phase 6.

1. Every type recorded in step 1 is still present, with its parent and connections unchanged. If one
   is missing or altered, you or a concurrent writer damaged the space's schema: HALT.
2. Every ADDED type appears in `types`.
3. Every ADDED type has a declared parent connection whose target exists in the merged schema.
4. The kind-enum audit below passes for every added type with a `kind` attribute of three or more
   values, and for every existing enum this run extended.
5. Structural sanity by inspection: the space still has exactly one root, and no added type has
   become a hub every other type attaches to.

### Kind-enum semantic audit (mandatory trace output)

Write the audit to `trace.md` BEFORE the phase 5 OK line. Format:

```
[HH:MM] phase 5 kind-audit - <Type>.kind:
  <value 1> -> natural parent: <Type> (reason: ...)
  <value 2> -> natural parent: <Type> (reason: ...)
  ...
  -> <N> distinct natural parents observed
  -> verdict: KEEP as one type | SPLIT into <list>
```

Apply BOTH criteria. Either one triggering SPLIT fails the audit.

**Criterion 1 - declared-parent uniformity.**

- All values share one natural parent type -> PASS.
- Values have two distinct natural parents -> PASS with a schema-debt note
  (`schema-debt: <Type>.kind has 2 natural parents, split if it grows`).
- Values have three or more distinct natural parents -> SPLIT. Do not advance. Register the split
  types first, then re-run the gate.

**Criterion 2 - value semantic distance.** Even when every value shares one declared parent, ask
whether the values name fundamentally different KINDS OF FACTS about that parent. Apply the SWOT
test: if three or more values map cleanly onto strength / weakness / opportunity / threat, or onto
otherwise non-overlapping analytical dimensions (risk, gap, theme, positioning, open question all
describe disjoint kinds of facts), the enum is a semantic catch-all and MUST split - even though it
is structurally uniform. If you cannot write one coherent sentence describing what the enum
represents, it is a catch-all.

KEEP is allowed when there is a single declared parent AND the values are facets of one coherent
dimension (employment-relationship facets of a person-like type) or operational sub-categories of one
process (launch / hire / partnership / funding as corporate-action events).

Worked example - an enum that looks innocuous but hides five parents in practice:

```
[HH:MM] phase 5 kind-audit - Note.kind:
  todo/milestone/blocker/deadline -> Project; observation -> root; complaint -> Service;
  idea -> Theme; owner -> Person; summary -> cross-cutting, no clear parent
  -> 5+ distinct natural parents -> verdict: SPLIT, one type per parent
```

Worked example - structurally uniform, fails on Criterion 2:

```
[HH:MM] phase 5 kind-audit - Topic.kind:
  {theme, risk, gap, opportunity, strength, weakness, positioning, open_question}
  -> declared parent: Company (all values structurally attach to a Company)
  -> criterion 1: PASS (1 declared parent)
  -> criterion 2: FAIL - three values map cleanly onto SWOT plus four disjoint analytical dimensions
  -> verdict: SPLIT into Theme, RiskSignal, FeatureGap, Opportunity, Strength, Weakness,
     Positioning, ResearchQuestion
```

A KEEP verdict is INVALID if the type name is on the banned-umbrella list. Both worked examples above
are umbrella-named; that is not a coincidence. Audit such a type as if the verdict were already
SPLIT, and use the audit to work out WHICH types to extract.

### On failure

Append `[HH:MM] phase 5 HALT - <what failed, which types>` and stop. Do not write instance data
against a schema that did not register. Surface the halt to the user with what you did and did not
change. If only structural sanity is off, fix it with `naumu_add_connection` or
`naumu_update_schema` and re-run the gate.

### Freeze the merged schema

On pass, write the MERGED schema to `schema.json` in the run dir: every type in the space after your
additions, with parents, connections, attributes and enums, each flagged as pre-existing or added by
this run. This file is the single source for the rest of the run:

- Phase 7 builds its edge-direction lookup from it, not from a fresh schema call per chunk.
- Fan-out workers read it and never call a schema-write tool.
- Mid-run schema evolution is the only thing that ever rewrites it: write `schema.json.tmp` and
  rename it over the original, and trace the rewrite. A resume never rewrites this file.
- Resume checks it: `naumu_get_schema` must still CONTAIN everything the file currently records,
  evolution included. The team may have legitimately added types since; only a missing or changed
  imported type halts.

Trace: `[HH:MM] phase 5 gate - <E> existing types intact, <A> added, kind-audit clean, schema.json frozen`
