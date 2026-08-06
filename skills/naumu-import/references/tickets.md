# Tickets and comment history

Read this in **Phase 7 Ingest**, and only if triage assigned any item to the
`ticket` tier.

A ticket export is two things stacked together: a set of records (the tickets
themselves, with keys, statuses, assignees, links) and a set of conversations (the
comment chains hanging off them). They import differently, so handle them as two
steps in that order.

**Transcript notes are the form for an imported archive.** A tracker that was
exported is a read-only historic artifact - nobody is going to reply to a 2023
ticket - and a note is the honest shape for that. It preserves the authors and their
original timestamps verbatim, keeps the comments in order, embeds the media where it
was posted, and can be read back and appended to, which is what makes a four-hour run
resumable. It is not a conversation and it does not pretend to be one.

## Ticket identity comes from the key, not the file

Derive every ticket's external key first (`PROJ-412`, `28257`) - from the record's id
field, the CSV column, the filename, or the saved page's heading. The key is the
dedup identity:

- Normalize the node label to `<KEY> - <title>` regardless of how the source
  formatted it, so the sibling-map key resolves every rendering of the same ticket to
  one node.
- Transform already folded multiple renderings of one ticket (a structured record and
  a saved page of it) into a single logical item by key. If you meet two items with
  the same key here, treat the richer one as canonical and note the other in the
  report - do not create two of anything.
- Author strings often carry role decorations (`Oleksandr K. [Projects manager]`).
  The Person node's label is the clean name; keep the decorated string verbatim in
  the transcript heading. The role can become an attribute when the corpus uses it
  consistently.
- A comment's own `attachments` and `images` arrays are AUTHORITATIVE wherever they
  exist. They say precisely which files belong to that comment, and nothing else needs
  consulting. The `att_<index>_<filename>` sibling convention is the last resort, for
  exports that carry no arrays at all - and that index is an export-wide counter, not
  a comment ordinal. Match those files by name or by the reference in the comment
  body. Never assume file N belongs to comment N.

## Step 1 - every ticket becomes a node (graph mode only)

Skip this step entirely in artifacts mode: no Issue nodes, no Person nodes from
ticket authors. The transcript note then opens with a metadata header (key, title,
status, dates, people) above the comment history, so the record survives without
touching the graph.

- One node per ticket, typed `Issue` (or whatever the target space already calls its
  issue type - Phase 4 Schema fit decided that; do not invent a second one).
- The label is the human-facing ticket title, prefixed with its key when the team uses
  keys: `PROJ-412 Sidebar drops unread state on reconnect`.
- Status, priority, created date, resolved date, reporter, assignee, and the source
  URL go on as attributes. Keep attribute names consistent across the whole export -
  the first ticket sets the convention for the rest.

  **Every one of these has to be registered on the Issue type before Phase 7 starts.**
  `naumu_add_node` validates attributes against the live schema and one unregistered
  key rejects the whole 25-node batch, so an unregistered `priority` costs the batch,
  not the field. `schema.md`'s Phase 4 attribute obligation owns this - the dates
  declared with type `"date"` and written `"YYYY-MM-DD"`, and any status or priority
  declared as a select with every value this export actually contains. If you meet a
  value the enum does not carry, normalize it locally to one that exists rather than
  letting the batch fail.
- Batch node creation in groups of at most 25.
- Every human named anywhere on the ticket (reporter, assignee, mentions, comment
  authors) becomes a Person node, following the same person discipline as the rest of
  the ingest in `ingest.md`. Check the sibling map before creating - the same reporter
  appears on hundreds of tickets and must resolve to one node.
- **The Issue node gets a parent edge like any other node**, not by implication: the
  connectivity gate in `ingest.md` step 4 applies to ticket nodes too, so each Issue is
  placed under the parent its type declares with `naumu_add_edge` carrying
  **`isParent: true`** - the epic where the ticket has one, otherwise the project or
  component hub the schema names as the Issue type's parent.
- Issue links between tickets - blocks, relates-to, duplicates - become mesh edges
  between the ticket nodes: no `isParent`, put through the edge-direction gate in
  `ingest.md` exactly like any other edge.
- An epic-to-ticket link is a parent relation, not a mesh edge. The ticket's parent edge
  to the epic node carries **`isParent: true`**, and it is the ticket's one parent edge -
  so an epic child does not also take a parent edge from the connectivity gate above.

**Tickets are enumerable entities, so all of them get nodes.** A 4,000-ticket export
produces 4,000 Issue nodes, not a summary of the interesting ones. The whole value of
importing a tracker is that the team can later ask about any ticket, including the
boring ones. Never substitute a digest for the records.

**A gated destination changes what the node may carry.** No topic hides a graph node,
so the node keeps the key, title, status, dates and edges while the gated transcript
note carries the comment bodies, the quoted emails, and any client-identifying prose.
SKILL.md's destination section owns that rule.

## Step 2 - the transcript note

Per ticket:

1. `naumu_create_note` titled `<KEY> - comment history`, carrying the destination's
   `topicIds` or `sharedWithSpace: true` like every note this run makes - never
   neither. Append the item's `artifacts.noteId` status line immediately, before a
   single comment goes into it.

   Where Phase 0 recorded `capabilities.noteBodyOnCreate`, a transcript whose comments
   carry no attachments rides along in that same call: build the whole thing from the
   staged ticket and pass it as `markdown`, one call for the note and its history, and
   step 2 has nothing left to do. Attachments cannot ride it - the presign is made
   against a `noteId` that only exists once the note does - so a ticket with embeds
   creates the note with whatever leading run of attachment-free comments it has and
   appends the rest below.
2. For each comment, in original order:
   - For each attachment on that comment: presign with the note's `noteId`
     (`naumu_request_attachment_upload({ graphId, noteId, fileName, fileType, fileSize })`),
     then PUT the exact bytes. Follow `attachments.md` for headers, ledger stages, and
     caps. An attachment over a cap is deferred and named in the report; the comment
     still lands without it.
   - Then make one `naumu_note_append` call carrying all of that comment:
     - a heading line `### <Author display name> - <original timestamp>`
     - the comment body as markdown
     - one sole-line `![alt](attachment://<attachmentId>)` per attachment you just
       uploaded for it

   A comment that carries embeds gets its own append: the ids have to ride in the same
   call as the text they belong to, and every id in that call must come from a presign
   made against this same note. Consecutive attachment-free comments may batch into one
   append, each keeping its own `### <author> - <timestamp>` heading. Never split a
   single comment across calls.
3. In graph mode, after the last comment, `naumu_update_node` on the ticket node to
   add `comment_count` and `transcript_note` attributes, so the node points at where
   its history lives.

What this does not preserve, and what to say once at the consent gate rather than
letting the user discover it after 4,000 tickets:

- Nobody can reply in a transcript note, it does not appear in the space's
  conversation list, and the authors are text rather than people.
- The link from the note back to the ticket node is soft - an attribute and a matching
  title, not a hard relation. Renaming the note breaks the trail.

## Living items - the exception

If the user themselves states that the items are still live - "this tracker is still
active, keep them as conversations" - then and only then does the thread path apply.
Never offer it, never suggest it, and never reach for it because an export looks
recent.

Cap it at roughly 50 tickets and state the costs before starting:

- Every message authors as your API identity, not as the original commenter. The
  original author survives only as text you write into the body.
- Every timestamp is ingest time, not the original time. The conversation reads as if
  it all happened today.
- It costs one background-agent invocation per ticket. At 50 that is tolerable; at
  500 it is a turn-budget trap that will run for hours and may not finish.

Per ticket, one `naumu_delegate` call. **The call itself creates the conversation and
returns that thread's id immediately, with `status: "processing"`.** @Naumu has not
opened anything yet - it starts working in the background on the thread the call just
made. So the thread you post into is the one in the response: take its `threadId`,
use it, and never wait for a thread to appear or ask for a second one. This is also
the only way a user session gets a thread at all; there is no direct
thread-creation tool on that surface, and the thread is a side effect of delegating.

The task text asks @Naumu to record this ticket's comment history and to relate the
thread to the ticket node. In gated mode the call carries the gated `topicIds`, so
the thread is born inside the topic. Then, on the returned `threadId`: presign each
comment attachment against it and PUT the bytes, then `naumu_post_message` with the
comment text and the returned ids in `attachmentIds`, at most 25 per message.

## Attachments hanging off comments

A comment's attachments are media items like any others, and the triage rubric applies
to each one before its bytes move:

- PDF or office document under 10MB - extract the text locally and fold a short
  extract into that comment's append, so something readable survives the upload.
- Image under 10MB - read it locally and write a real one-line caption. That read is
  the last moment the pixels are legible to anyone.
- Over a cap either way - defer it with the right reason and name it in `report.md`.
  A 30MB mockup on a comment is over the agent-read cap, which no plan upgrade lifts.
  The comment still lands without it.

Ticket-owned media counts toward the consent gate's upload total like every other byte
this run moves.

- The transcript path presigns against the note's `noteId` and binds with the
  `naumu_note_append` that carries the comment. The thread exception presigns against
  the `threadId` and binds with the `naumu_post_message`.
- Alt text follows `attachments.md`: a real description for images you read locally,
  the file name for anything you did not.
- Every upload writes its `presigned`, `put_ok`, `bound` rows to `uploads.jsonl` like
  any other. Re-embedding an already-bound id in a later comment is safe and needs no
  new ledger row; a resume finds what already landed with `naumu_note_read`, not from
  the ledger.

## Resume

Ticket items are ledger items like everything else: `in_progress` into `status.jsonl`
before the ticket's first write, `done` after its last.

Recovering one:

- Step 1 is cheap to re-run. The sibling map resolves the ticket key to the node that
  already exists, so re-running produces enrichment rather than a duplicate.
- Call `naumu_note_read` on the transcript note. The headings tell you exactly which
  comments already landed, and embeds round-trip back to the same
  `![alt](attachment://<id>)` syntax so you can see which files are bound. Append only
  the missing tail, in order. Do not rewrite the note and do not re-upload anything the
  ledger or the note already shows bound.
- On the thread exception, read the thread back and post only the comments that are
  missing.
- Re-running the closing `naumu_update_node` is harmless; let it correct
  `comment_count` on the way out.
