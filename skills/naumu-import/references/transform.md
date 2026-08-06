# Transform

Read this in **Phase 2 Transform**, after the survey has written a manifest line for
every raw file.

Raw files are not the unit of work. A browser-saved article arrives as twenty
files; a support ticket arrives as a JSON plus a folder of attachment siblings.
Transform folds those piles into **logical items** - one article, one ticket, one
document - and materializes a parse-friendly copy of each under `staging/`.

Transform writes nothing to Naumu. It reads the corpus, writes `staging/`, and
appends manifest lines.

**Transform is script work, not reading work.** For a corpus of many similar
sources - hundreds of saved pages, a folder of per-ticket exports - write the
normalization as a deterministic script inside the run directory (a
`transform.mjs`, a Python file, whatever runs here) and execute it once. Corpus
bodies do not pass through your context: read a bounded sample per bucket to
calibrate the script, run it, then read a sample of its OUTPUT to verify. The
script doubles as the reproducibility story - "delete staging/ and get the same
thing back" means re-running it. Hand-transforming file by file is only
justified when the corpus is genuinely small or each source needs individual
judgment.

## staging/ is derived and disposable

The raw corpus is read-only truth. `staging/` is a derived view of it: an index of
what became what, plus the normalized content itself.

- Delete `staging/` and re-run Phase 2 and you must get the same thing back. Nothing
  downstream may depend on a staged file that transform could not reproduce from the
  raw corpus.
- **Never copy media bytes into staging.** Images, PDFs, video and audio are
  referenced by their path under the corpus root, and the logical item that owns each
  one is recorded. Uploads always read from the original file, never from a copy.
- Nothing in `staging/` is ever uploaded. It exists so later phases can parse cheaply.

That is why a resumed run can rebuild staging without asking anyone: it is not
state, it is a cache with a deterministic recipe.

## Normalize, then bucket

For each text-like source, write one staged artifact carrying the content worth
reading and nothing else:

- Strip what carries no meaning: navigation, scripts, style blocks, cookie banners,
  footer chrome, mail-client quoting boilerplate, export headers.
- Keep the body, its headings, its tables, its code blocks, and its references to
  media by path.
- Keep the source's own metadata where it exists - title, author, timestamps, ticket
  key, status, source URL - so triage and ingest do not have to re-derive it.

Sort the results into **kind buckets** under `staging/`: `tickets/`, `articles/`,
`docs/`, and whatever else this corpus actually contains. The bucket is a coarse
statement about what a thing is, and it is what `todo.md` counts.

## Collapsing a pile into one item

**A browser-saved page is one article.** It arrives as `<name>.html` plus a
`<name>_files/` directory. The `.html` is the content: render it to text, strip the
chrome, keep the article body. Scripts, styles, fonts, sourcemaps and
`saved_resource*.html` inside `_files/` are noise.

The article's owned media is what the RETAINED body still references once the chrome
is gone. Those images and PDFs stay raw, the staged article owns them, and each one is
recorded with the point in the body where it was referenced - so ingest can place the
embed where the author put it, with `naumu_note_insert` or
`naumu_note_replace_section` once the note has structure, and a plain append only when
the append order already matches the source. A file referenced solely from a stripped
region - the site header, the navigation, an avatar, a theme asset - is noise, even
sitting inside `_files/`.

**A per-ticket export is one ticket.** A JSON holding a ticket id, a source system
and a comment array, plus sibling files named by convention (`att_<index>_<file>`),
is a single logical item. The staged ticket carries the metadata, the comments in
order, and each comment's attachment references by path. The `att_*` siblings are not
independent media items; they belong to their comment.

**The same thing exported twice is still one item.** Exports routinely carry a ticket
as a structured JSON, a row in a CSV, and a saved HTML page of it, often in three
different buckets. So before minting any ticket item, build a ticket-key index across
ALL buckets. The keys are derived here, while parsing - Phase 1 stays body-blind and
cannot see them.

Where one key has several renderings, keep the richest as the item: structured JSON
beats CSV beats saved HTML. Mark every other rendering `skip` with reason
`duplicate-rendering` and a `supersededBy: <itemId>` field naming the survivor. That
field is not `dupOf`: `dupOf` means identical bytes, while these are different bytes
saying the same thing. The canonical label and title come from the surviving rendering
alone, never stitched together across renderings.

## Raw noise is skipped here

Triage never sees these. Skip a raw file, with its reason, when it is:

- **noise** - anything under `.git/`, `node_modules/`, `__MACOSX/`, plus `.DS_Store`,
  `*.lock`, `*.pyc`, editor and OS scratch files, and the script/style/font/sourcemap
  assets inside a saved page's `_files/` directory;
- **empty** - zero bytes;
- **a byte duplicate** - the same `sha256` already seen in this manifest. Record the
  first item's id as the duplicate's origin.

Everything else either becomes a logical item, is consumed by one, or survives as a
raw file for triage to tier (that is how loose media reaches Phase 3).

## What transform appends to the manifest

Two kinds of line, both appended, both following the record shapes in `state.md`:

- **A logical item line** - its own id, its bucket, its staged path, and the raw ids
  it was built from.
- **A revised raw line** for every raw file that was consumed, pointing at the logical
  item it now belongs to, or carrying a `skip` tier and one of the reasons above.

```json
{"id":"L_4c81a09e","logical":true,"bucket":"tickets","staged":"staging/tickets/28257.json","sources":["9f2a41c7b0de","a71c93f0d2b8"],"ts":"..."}
{"id":"9f2a41c7b0de","path":"export/28257/ticket.json","partOf":"L_4c81a09e","ts":"..."}
```

The logical id is derived, not invented: `L_` plus the first 12 hex of the sha256 of
the item's source raw ids, sorted and joined with newlines. Same sources, same id,
every rebuild - that is what lets `staging/` be disposable and keeps the ledgers
stable across a resume.

`sources` lists only the raw files the surviving item was actually built from - here
the ticket JSON and its attachment sibling. A rendering that lost the fold is not a
source; it is its own skipped line carrying `supersededBy`.

From Phase 3 on, **the logical item is the unit of work**: it is what gets a tier, a
status, an artifact id, a resume decision, and a fan-out shard. Raw files that were
consumed are no longer worked on directly; they are read through the item that owns
them.

## When a source will not parse

Leave it raw. Append a manifest line marking it unparsed with a short reason, let
Phase 3 tier it as the file it is (usually `note-only` or `attach-only`), and name it
in `report.md`. One malformed export does not stop a four-hour run, and a file that
lands as an opaque attachment is better than a file that does not land at all.

```
[HH:MM] phase 2 unparsed - export/legacy/dump.xml: no recognizable record structure, left raw
```

## todo.md

`status.jsonl` is the machine truth. `todo.md` is the human one: a compact checklist,
regenerated from the ledgers at every phase boundary, that a person can read in ten
seconds.

```markdown
# acme-export - phase 7 ingest

- articles   812/812 transformed, 812/812 triaged, 540/812 ingested
- tickets    178/178 transformed, 178/178 triaged,  90/178 ingested
- docs        96/96  transformed,  96/96  triaged,  96/96  ingested
- media     1,946 files, 1,204 uploaded, 37 deferred (over cap)
- skipped   2,914 noise, 3 unparsed
```

Two rules keep it honest:

- It is **regenerated**, never edited in place. Every line comes from folding the
  ledgers.
- It is **never read back as state.** No decision anywhere in this skill is made from
  `todo.md`. If it disagrees with `status.jsonl`, the ledger is right and the view is
  stale.

## Yours to decide

- The staged artifact format - one JSON per item, markdown chunks, or whatever parses
  most cheaply for this corpus. Be consistent within a bucket.
- The bucket taxonomy beyond the obvious ones, and how deep it nests.
- How often `todo.md` is regenerated beyond the phase boundaries it must cover.

```
[HH:MM] phase 2 transform - 812 articles, 178 tickets, 96 docs staged from 4,812 raw files; 2,914 noise skipped, 3 unparsed
```
