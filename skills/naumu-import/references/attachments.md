# Attachments

Read this in **Phase 7 Ingest**, and only if triage assigned any item to
`attach-caption` or `attach-only`, or a `note-only` item carries a file to embed.

Uploading sends the file's bytes to the team's Naumu storage, where every member of
the space can see them. That is the point, and it is also why the consent gate states
the upload total before anything moves. That total is an estimate: if the actual
uploads would run more than 20% past it, pause, say by how much, and re-ask before
continuing.

Bytes always come from the original file under the corpus root. `staging/` holds
normalized text, never media, so nothing here ever uploads a staged copy.

Caps and tier assignment are decided in triage, not here; `triage.md` owns both
ceilings.

## The shape both flows share

Every upload is presign, PUT, bind. The presign call is
`naumu_request_attachment_upload` with exactly one destination field, and the bind
call is whatever tool belongs to that destination. An upload is keyed to the
destination it was presigned for; you cannot presign against one note and bind
against another.

There are two destinations and no others. There is no canvas path in this skill.

The PUT is the same everywhere:

- `method` is `PUT`. Send the whole file as one fixed-length body.
- Send every header in `requiredHeaders`. It carries both `Content-Type` and
  `Content-Length`, and `Content-Length` is signed into the URL - so the body has
  to be exactly the `fileSize` you declared. A different byte count answers 403.
- Do not stream it and do not use chunked transfer encoding.
- Do not add an Authorization header. The URL itself is the auth.
- Do not log `uploadUrl` anywhere, including `trace.md`. It is a bearer capability
  until it expires.
- `expiresAt` is a Unix-ms timestamp. Bind before then or the upload orphans.

## Note flow - the default

Almost all imported media lands here, because a user session cannot open new
conversations and notes are the destination that always exists.

The note is also the access boundary. Every note this run creates carries either the
destination's `topicIds` or `sharedWithSpace: true`, never neither; SKILL.md's
destination section owns that rule. It matters here because a note created with the
gated `topicIds` gates everything embedded in it - non-members do not see the note, do
not see its files in the space's Files and Media library, and cannot fetch the bytes.
That work happens at `naumu_create_note` time, so the upload flow below needs no extra
parameters.

1. `naumu_request_attachment_upload({ graphId, noteId, fileName, fileType, fileSize })`.
   Returns `attachmentId`, `uploadUrl`, `method`, `requiredHeaders`, `expiresAt`.

   For audio, add `audioDurationSec` whenever the source actually reports the length.
   Audio carries an 8-hour ceiling on top of its size cap, nothing measures the file
   server-side, and the ceiling is checked only against a duration you supply -
   omitting the field is legal and simply skips that check. Omit it rather than
   guessing; a wrong value is worse than none, and a deliberate under-report is a
   policy violation, not a workaround. An over-length presign is refused with **400**,
   deliberately distinct from the size **413**, because the remedies differ: trim the
   recording versus send fewer bytes or upgrade the plan. Treat both as permanent for
   that file - `triage.md` owns the deferral reasons.
2. PUT the exact bytes as above.
3. Call a note write tool whose markdown contains
   `![alt](attachment://<attachmentId>)` on a line of its own -
   `naumu_note_append` for the normal additive case, or `naumu_note_insert`,
   `naumu_note_replace_section`, or `naumu_note_replace` when the import is
   placing media inside existing note structure.

That one write does three things at once: it binds the upload, it embeds the file
as the note's canonical inline media node (image, video, audio, or generic file,
dispatched by the upload's MIME type), and it files the attachment into the space's
Files and Media library. There is no fourth call.

Rules worth holding onto:

- The embed must be a **sole-line image**. An `attachment://` reference inline in a
  paragraph is not the same thing and will not bind.
- The id has to come from a presign made with the **same** `noteId`. A wrong-note,
  expired, or foreign id fails the **whole** write with a 400 that names the bad
  ids - the other blocks in that call do not land either. Batch embeds carefully
  and keep each note write to ids you presigned in the same pass.
- Re-embedding an id that is already bound in that note is safe and idempotent. No
  re-upload happens. This is what makes a resumed session cheap.
- `naumu_note_read` round-trips embeds back to the same `![alt](attachment://<id>)`
  syntax. Use it on resume to see exactly which files already landed before
  appending anything.

## Thread flow - only where a thread already exists

Use this when the user pointed at a conversation, or when a ticket path produced a
thread to record history into. Never as the general destination for media.

1. `naumu_request_attachment_upload({ graphId, threadId, fileName, fileType, fileSize })`.
2. PUT the exact bytes as above.
3. `naumu_post_message` with the returned id in `attachmentIds`. That call binds
   the upload. A single message carries at most 25 attachment ids; split larger
   sets across messages.

The presign only checks that the thread exists. Participation is enforced later, at
bind time, when `naumu_post_message` runs - so a presign that succeeds is not proof
you may post. Get the access question settled before uploading bytes, not after.

## Nodes hold no files

There is no node-attachment flow, and that is a design decision rather than a gap.
A graph node carries attributes and edges. The supported way to give a node media
is to put the file where the work happened - a thread that originated or modified
the node, or the note that owns the node - and let it surface on the node through
those threads and notes. When the import wants "the diagram for this component",
the answer is the component's note, embedded there, with the node edged to it.

## The uploads ledger

Every upload writes to `uploads.jsonl`, keyed by the full sha256 of the bytes plus the
destination kind and id, staged `presigned` **before** the PUT, then `put_ok`, then
`bound`. `state.md` owns the write stages and the resume table; follow it there rather
than reconstructing the rules from memory.

One cross-check belongs to this file. For note destinations, read the note back with
`naumu_note_read` before appending: if the embed is already there, the row is `bound`
in fact even when the ledger line never got written. Re-embedding is idempotent, so
when the note and the ledger disagree, include the embed and let the note deduplicate.

## Alt text

Alt text is the only description of the file that any future reader or agent gets,
so it carries real weight.

- For `attach-caption` items, write a genuine one-line description from the local
  read: what the image actually shows, in plain words. "Deployment topology with
  the queue between the API and the workers" is useful. "Screenshot" is not.
- For `attach-only` items, use the file name. You did not read the bytes and you
  must not invent what is in them.
