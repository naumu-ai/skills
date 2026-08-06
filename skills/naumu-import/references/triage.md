# Triage

Read this once, in **Phase 3 Triage**, after transform has staged the corpus into
logical items.

Triage gives every item exactly one tier. The tier decides what gets read locally,
what bytes leave the machine, and where the result lands in the space. Triage itself
writes nothing to Naumu and uploads nothing - it only revises manifest lines (last
line wins per `id`).

Its subjects are the **logical items** transform produced, plus the raw files no
logical item consumed (loose media, mostly). Noise, empty files and byte duplicates
were already skipped in Phase 2 and never reach here.

## Three rules that shape every decision below

**1. The 10MB agent-read cap is flat, and no plan lifts it.**
Any file type an agent reads whole - image, text, PDF, office document - is capped
at 10MB on every plan. Upgrading the space does not move that number. The plan
umbrella (the bigger per-space ceiling) applies only to video, audio, and archives,
which are never handed back whole. When you defer a 14MB PDF, say plainly that an
upgrade will not help; the fix is to split or convert the file locally.

**2. Nothing can read an attachment back.**
Once bytes are uploaded they are storage, not context. No tool returns their
content to an agent. So every fact the graph will ever carry about a file has to be
extracted from the LOCAL copy BEFORE the upload. That is the whole reason
`attach-caption` exists as a separate tier: the local read is the only moment the
pixels are legible, so alt text - and any entities the image yields - get written
then or never.

**3. Notes are the default destination for media.**
A user session cannot open new conversations in the space. Media goes to a thread
only where a suitable thread ALREADY exists (one the user pointed at, or one a
ticket path produced). Everywhere else it lands in a note. Plan for notes, treat
threads as the exception.

The plan cap that rules 3 and 8 need is probed once in Phase 0. If that probe came
back `unknown`, the conservative floor is concrete: treat the plan ceiling as
**50MB** - what the smallest plan allows for video, audio and archives - and defer
anything over it with `reason: plan-cap-unknown`, saying so in the report rather
than guessing generously. (The 10MB agent-readable cap is not affected; it is the
same on every plan.) Those deferrals are exactly the ones a later run with a
working probe can lift.

## Triage is script work, like transform

The rubric below is deterministic. Mime type, byte size, staging bucket, and the
owner-resolution ladder are all path and metadata logic over `manifest.jsonl` - no
corpus body has to be read to run any of it. So run it the way transform runs:
**write the rubric as a script in the run directory and execute it over the
manifest**, rather than adjudicating a thousand manifest lines one at a time in the
conversation.

The same contract transform works under applies here:

- Calibrate on a bounded sample per bucket before writing the script - twenty lines
  from each, enough to see what the mime types and sizes actually look like in this
  corpus.
- Run it once. It appends the revised manifest lines itself, in the shape below.
- Verify on the OUTPUT: read a sample per resulting tier and check the tier a human
  would have given it, plus the counts per tier against the survey totals.
- Hand-adjudicate only the ambiguous residue - the "canonical long-form artifact"
  judgment in rubric rule 5, an unknown mime the script could not place, a media file
  the owner ladder could not settle. That residue is normally tens of items, not
  thousands.

Corpus bodies and per-item reasoning stay out of the conversation either way. The
script is also the reproducibility story: re-run it and get the same tiers back.

## The rubric - ordered, first match wins

Walk the list top to bottom for each item and stop at the first rule that matches.
Record `tier` and a short `reason` on the manifest line.

1. **A staged item in a ticket bucket** - transform recognized a ticket or chat
   export shape (a per-ticket record with a comment array, an issue key, an
   `issues/` or `tickets/` export) -> **ticket**.
   First match wins, so this rule sits above the 10MB branch and would otherwise
   hand `tickets.md` an item nothing can read whole. Check the size here: a staged
   ticket item over 10MB stays `ticket` but carries `read: sectioned` on its
   manifest line, and Phase 7 walks it by comment range rather than reading the
   file in one go. A ticket export is usually a long array of small records, so
   sectioning is cheap; deferring it would drop hundreds of tickets over one file
   boundary.
2. **Agent-readable AND over 10MB.** Text-like (markdown, plain text, source, large
   JSON or CSV that is not a ticket export) -> **graph-extract** through sectioned
   local reads, upload disabled for that item. Image, PDF, or office document ->
   **defer** (`reason: over-agent-read-cap`). Say in the report that this ceiling is
   not liftable by plan.
3. **Not agent-readable AND over the plan cap** (video, audio, archive) -> **defer**
   (`reason: over-plan-cap`). Say in the report that this one IS liftable by
   upgrading the space's plan.
4. **Text under roughly 200KB** -> **graph-extract**. Read it, pull nodes and edges,
   no note.
5. **Text between roughly 200KB and 10MB, or any canonical long-form artifact** (a
   spec, a handbook, a policy, a help-center article, a transcript the team treats as
   a document) -> **note-only**. The note carries the text verbatim; in graph mode
   one Document-typed node represents it in the graph.
6. **PDF or office document under 10MB** -> **note-only**. Read it locally, write an
   extracted-text summary into the note, embed the file inline, and pull entities
   from the first pages rather than the whole thing.
7. **Image under 10MB** -> **attach-caption**. Read it locally, write real alt text,
   upload, embed it in the owning note.
8. **Video, audio, archive, or unknown type under the plan cap** -> **attach-only**.
   Upload and embed with no local read. Audio is worth keeping even when long: it is
   transcribed on the way in, and the transcript is what reaches an agent later.

   **Audio carries a second ceiling: 8 hours, on top of the size cap.** Nothing
   measures the file server-side, so the ceiling is checked only against a duration
   you report. `naumu_request_attachment_upload` takes an optional
   `audioDurationSec`; send it whenever the source actually tells you the length
   (an export's metadata, a sidecar, a container header you can read cheaply).
   Omitting the field is legal and simply skips the check - the size cap still
   binds - so omit it rather than guessing. A wrong value is worse than none.
   Anything you already know runs past 8 hours is **defer** with
   `reason: over-audio-duration-cap`, named in `report.md` like any other deferral;
   the fix is splitting the recording locally, and no plan upgrade lifts it.

   A duration rejection comes back as a **400**, deliberately distinct from the
   size **413**, because the remedies differ - trim the recording versus send fewer
   bytes or upgrade the plan. `state.md`'s never-retry-a-413 rule does not cover
   it; treat a 400 on presign the same way, as a permanent refusal of that file,
   and never retry it with a shaved-down duration.
9. **Anything left** -> **defer** (`reason: unreadable`).

`skip` stays available for the rare item transform let through that turns out to
carry nothing, but after Phase 2 most skipping is already done.

**Artifacts mode applies one override immediately after the rubric, and before
anything downstream reads a tier**: `graph-extract` items are demoted to `note-only`
(their entities stay in the note's prose instead of becoming nodes), `ticket` items
keep their tier but produce no nodes - the ticket pipeline runs in its notes-only
shape (see `tickets.md`) - and `note-only` items produce no Document node, because in
artifacts mode a note is the entire landing. Nothing else changes; the rubric itself
is mode-agnostic.

**Re-triage in an enrichment pass.** When an artifacts-only run is continued as a
post-import graph enrichment pass, Phase 3 runs a second time with
`destination.mode` now `graph`, so the override above no longer applies. It decides
exactly one thing: which items are `graph-extract` eligible. Items already terminal
as artifacts KEEP every artifact they produced - nothing is re-noted, re-uploaded or
re-owned, and the owner-resolution ladder below does not run again. In practice the
demoted items go back to the tier rule 4 or 5 would have given them, `ticket` items
regain their node half, and `note-only` items gain the Document node graph mode
requires. Append the revised lines the same way as any other triage pass: last line
wins per `id`, so the enrichment tier is simply the newest line.

Ordering matters here. Owner resolution below carves out `graph-extract` items
because they produce no note to own anything - but in artifacts mode they DO produce
one. Apply the demotion first and resolve owners against post-override tiers.
Resolving first would push media whose only referencing item became a note down to
rule 4 or to loose media, filing it in a directory note while the note that actually
talks about it sits right there.

## Where each tier lands

| Tier | Local read | Bytes uploaded | Lands in the space as |
|------|-----------|----------------|------------------------|
| `graph-extract` | full or sectioned | no | nodes and edges only |
| `note-only` | full (or first pages for PDF/office) | only for PDF/office | a note with the text, plus - graph mode only - one Document-typed node parented to the home Phase 4 chose for the class |
| `attach-caption` | yes, once, before upload | yes | an inline embed in the owning note, with agent-written alt text |
| `attach-only` | no | yes | an inline embed in the owning note, alt text = file name |
| `ticket` | full | per-comment attachments only | a transcript note, plus an Issue-typed node in graph mode |
| `skip` | no | no | nothing |
| `defer` | no | no | nothing, but named in `report.md` with its reason |

That Document node is created in Phase 7, at the same time as its note, and parented
directly to the home Phase 4 found for the `note-only` class. It is NOT edged into a
source mini-hub at creation - no mini-hub exists yet. Phase 9a groups the run's
Document nodes into source mini-hubs afterwards, once they all exist; see
`restructure.md`.

Deferred and skipped items are not failures to hide. Every one of them gets a line
in `report.md` with its path, size, and reason, so the team can decide whether to
convert, split, or upgrade.

## Deciding which note owns a media file

Both media tiers embed into an "owning note", so triage has to name one. Every tier
named below is the POST-override tier: in artifacts mode the demotion has already
run, so nothing here is still `graph-extract`. Resolve in this order and stop at the
first hit:

1. A logical item references the file by path (a markdown image link, an HTML `src`,
   a relative path in a spec) -> that item's note owns it. Transform already recorded
   most of these. The exception is a `graph-extract` item, which produces no note at
   all: fall through to rule 4, and then to loose media.
2. The file is an attachment on a ticket or one of its comments -> that ticket's
   transcript note owns it. See `tickets.md`.
3. The file shares a basename with a sibling document (`architecture.png` next to
   `architecture.md`) -> that document's note owns it.
4. Exactly one `note-only` item sits in the same directory -> that note owns it.
5. Nothing above matches -> it is loose media, handled below.

Record the resolved owner on the manifest line. Ordering in Phase 6 has to put the
owning note's item before anything that embeds into it, because the presign needs a
real `noteId`.

## Loose media with no owning document

Most media arrives next to something that owns it - a spec, a ticket, a folder of
prose. That owner's note is the destination.

For media with no owner, do NOT create a note per file. One note per directory
instead, created lazily in Phase 7 at that directory's first upload - triage writes
nothing to Naumu, it only names the owner. Title it `Media - <relative dir>`, with the
path relative to the corpus root, and give it the run's audience like every note this
run makes: the destination's `topicIds` for a topic import, `sharedWithSpace: true`
for a space-wide one, never neither.

That note gets its own synthetic item id - `M_` plus the first 12 hex of the sha256 of
the relative directory - so it has a subject in the ledgers. Triage records only the
OWNER directory on each orphan's manifest line; Phase 7 records the note id on the
synthetic item's first status line, and every later orphan from that directory appends
into the same note.

A directory with a single stray screenshot gets one note with one image in it. That
is fine. A directory with 300 screenshots gets one note with 300 embeds, which is
what a human would have made.

## Writing triage results

Append one revised manifest line per item, carrying at least `id`, `tier`, `reason`,
and `dupOf` where it applies. Do not rewrite earlier lines; the fold reads
last-line-wins, so a corrected judgment is just another append.

Close the phase with a trace line, regenerate `todo.md`, and give the user a
one-screen summary: counts per tier, total bytes that will be uploaded if they say
yes, and the deferred list. In artifacts mode that summary is what the consent gate
shows, since Phase 3 is the last read-only phase; in graph mode it carries into the
gate at the close of Phase 4. Either way, keep it honest and keep it short.
