# Nebulanote Data Model

> Status: draft. Describes the **logical** data model. Storage engine,
> sync protocol, indexing, and persistence format are deliberately
> deferred — see *Open Questions*.

## Scope

This document defines the entities, fields, and relationships that make
up a user's Nebulanote data. It does **not** pick a database, a file
format, or a sync mechanism. Those will be separate documents once the
shape is settled.

Models are written as pydantic skeletons for clarity. They are
design-time schemas, not runtime code — no validators, no methods, just
types and inline comments.

## Design principles

1. **Multi-device by birth.** Every record gets a **UUIDv7** at the
   moment it is created on any device. UUIDv7 encodes a millisecond
   creation timestamp in its high bits plus randomness in the low
   bits, so ids are globally unique across devices *and* sort
   lexicographically by creation time. No sequential or
   device-dependent IDs.
2. **Two kinds of records, one stream.** A **Note** is the user's
   expression (what they wrote); a **Resource** is the file-shaped
   thing they captured or collected (photo, voice memo, video, PDF,
   …). Notes reference Resources by id; Resources live on their own
   and can be referenced by many Notes.
3. **Large content lives outside records.** Notes and Resources are
   small structured records. Actual file bytes live in a separate
   content-addressed blob store and are referenced by hash.
4. **Content-addressed blobs.** Files are identified by the hash of
   their bytes (SHA-256). Same bytes → same ID, so uploading the same
   photo twice costs nothing extra, and syncing can skip blobs another
   device already has.
5. **Soft delete.** Deletion is marked, not erased, so it can propagate
   to other devices during merge. Actual garbage collection of
   tombstones and orphaned blobs is a separate later concern.
6. **Minimal required fields.** Capture is supposed to be effortless,
   so the bare minimum for a valid note is very small.

## Key decisions

- **Two top-level entities: `Note` and `Resource`.** A Note is what
  the user wrote; a Resource is a file-shaped thing they captured
  (photo, voice memo, video, document). Notes reference Resources by
  id; Resources stand on their own and can be referenced by many
  Notes.
- **A Note's content is a single string with inline Resource refs.**
  The note has one `content` field. Resources are referenced inline
  via markers of the form `{{kind:<resource-uuid>}}` (e.g.
  `{{image:abc…}}`, `{{voice:def…}}`). This makes ordering and
  interleaving expressible by the user (*"at the gym
  `{{image:before}}` did squats `{{voice:grunt}}` `{{image:after}}`"*),
  mirrors how Markdown / Obsidian / Notion already work, and keeps
  the Note schema tiny.
- **The Note does not carry an attachments list.** The set of
  Resources a Note references is derived by parsing its content
  string. Storage-level indexes for reverse lookup (*"which notes
  reference this Resource?"*) are a later concern, not a data-model
  field.
- **Content must be non-empty.** Empty notes are not meaningful. A
  voice-only note is valid — its `content` is just the single marker.
- **Capture creates both a Resource and a Note.** Taking a photo or
  recording a voice memo produces a Resource *and* a minimal Note
  containing only its marker. This keeps the stream complete and
  avoids creating orphans at capture time.
- **Every inline ref must resolve** to a live Resource. Dangling refs
  are invalid; the `kind` in the marker must match the Resource's
  `kind`.
- **Resources should have at least one referencing live Note.**
  Resources whose referencing notes all become deleted are flagged as
  orphan cleanup candidates. They are *not* auto-deleted — user
  reviews and decides.
- **Resource metadata is partly typed, partly free-form.** A small
  core is typed (`kind`, `mime`, `blob_hash`, `size_bytes`,
  timestamps). Everything kind-specific (dimensions, duration, EXIF,
  OCR text, AI-generated descriptions, page counts, transcripts, …)
  lives in a `meta: dict[str, Any]` field, with per-kind conventions
  documented in prose rather than forced into the type system.
- **Tags are plain strings on the note.** Normalized to lowercase, no
  hierarchy, free-form. Simple and fast. Added manually by the user
  or suggested by AI.
- **Blobs are content-addressed.** A separate blob store holds file
  bytes keyed by SHA-256. Resources reference blobs by hash only.

## Core entities

### Note

The user's expression. A single `content` string that interleaves
plain text with inline references to Resources.

```python
class Note(BaseModel):
    id: UUID  # UUIDv7 — sortable by creation time, globally unique
    created_at: datetime  # when the note was captured, on the source device
    updated_at: datetime  # last edit time; equals created_at if never edited
    deleted_at: datetime | None  # soft-delete tombstone; None means live

    content: str  # free-form text with inline refs like {{image:<resource-uuid>}}
    tags: list[str]  # lowercase tag strings, unique within the note

    source_device: str | None  # short label of the device that captured this
```

Notes are ordered by `created_at` in the stream view (equivalent to
ordering by `id` thanks to UUIDv7). `updated_at` is the hook a future
sync mechanism can use for last-writer-wins merge; `deleted_at` is
the corresponding hook for propagating deletions.

### Content and inline references

The `content` string is the single source of truth for what the note
says, what Resources it uses, and in what order. Resources are woven
into the narrative by reference, not by position in a list.

**Marker syntax** (working choice, see Open Questions):

```
{{kind:<resource-uuid>}}
```

Where `kind` is one of `image`, `voice`, `video`, `file`, matching
the referenced Resource's `kind`. Examples:

```
Trying a new recipe today {{image:3f1a…}} — smelled great.
{{voice:9b2c…}}
Test results from today {{file:1d4e…}} will chat with the doctor tomorrow.
```

**Invariants:**

- Every marker in `content` must resolve to a live Resource with the
  same `id`.
- `kind` in the marker must match the Resource's `kind`. Mismatch is
  invalid (protects against accidental wrong-file renders).

**Escaping:** how to include a literal `{{…}}` in text without it
being parsed as a marker is deferred. Most users will never hit this;
when they do we add an escape rule.

### Resource

A file-shaped thing in the user's archive: photo, voice memo, video,
document. A Resource has identity and a life of its own independent of
any particular Note — the same Resource can be referenced from many
Notes (today's entry, a weekly summary, a yearly retrospective).

```python
class Resource(BaseModel):
    id: UUID  # UUIDv7 — used in note markers like {{image:<id>}}
    created_at: datetime  # when the resource was added to the system
    updated_at: datetime  # last time meta was updated (e.g. AI enrichment)
    deleted_at: datetime | None  # soft-delete tombstone; None means live

    kind: Literal["image", "audio", "video", "document"]  # coarse class
    mime: str  # authoritative MIME type, e.g. "image/jpeg"
    blob_hash: str  # SHA-256 hex of the file bytes; key into the blob store
    size_bytes: int  # file size in bytes

    meta: dict[str, Any]  # kind-specific metadata; see conventions below

    source_device: str | None  # short label of the device that captured this
```

`meta` is deliberately free-form `dict[str, Any]` because what lives
in it varies per kind and will grow as features (AI analysis, OCR,
transcription, …) are added. Conventions, not types:

- **image** — `width`, `height`, `exif` (sub-dict), `ai_description`,
  `ai_tags`, `filename`.
- **audio** — `duration_ms`, `transcript`, `ai_summary`, `filename`.
- **video** — `duration_ms`, `width`, `height`, `transcript`,
  `ai_summary`, `filename`.
- **document** — `filename`, `pages`, `ocr_text`, `ai_summary`.

Any of these may be absent. AI-derived fields appear only after an
analysis runs. Trade-off noted: `dict[str, Any]` gives up mypy
coverage over meta in exchange for evolvability; we can promote hot
keys to typed sub-models later if warranted.

### Capture and resource lifecycle

- **On capture**, the client creates a Resource *and* a minimal Note
  whose `content` is just the single marker `{{kind:<resource-id>}}`.
  The stream always contains every captured thing; orphan Resources
  only arise later.
- **Resources may be re-referenced.** When the user writes a later
  Note, they can include `{{image:<old-resource-id>}}` to weave in a
  previously captured photo. No copy is made.
- **Cascade on Note delete.** Soft-deleting a Note has no immediate
  effect on Resources. A Resource whose every referencing Note is
  deleted becomes an **orphan candidate**, surfaced to the user for
  review. Orphans are not auto-deleted.
- **Temporal interpretation.** `Resource.created_at` is when the file
  entered the system; `Note.created_at` is when the writing happened.
  A recent Note may reference an old Resource; UIs must not conflate
  the two timestamps.

### Blob (storage concept, not a modeled record)

A blob is just file bytes addressed by their SHA-256 hash. Blobs are
not a pydantic model because they are not a structured record — they
live in a separate blob store (filesystem, object store, or whatever
we pick later). Resources reference blobs exclusively through
`Resource.blob_hash`.

Two consequences worth noting:

- If the same file is added twice (same bytes), the two Resource
  records both point to the same blob — no duplication of bytes.
- Deleting a Resource does not delete the blob directly. Blob garbage
  collection runs separately once no live Resource references a
  blob.

## What we are not modeling yet

Deferred to later documents; listed here so we notice what is missing.

- **Sync metadata.** Device IDs, vector clocks, last-sync timestamps,
  merge journals. The data model leaves hooks (UUIDs, `updated_at`,
  `deleted_at`) but does not specify a mechanism.
- **Groups / curated notes.** The vision describes collapsing raw
  entries into a refined summary. Will be modeled once the flow is
  understood — candidates include "a note that references other notes"
  or a dedicated `Group` entity.
- **Sharing state.** Public / password-protected visibility of
  individual notes. Needs its own record (share links, passwords,
  expirations).
- **AI-derived data shape.** We've committed to "AI output lives in
  `Resource.meta`" as a home, but not to the exact keys, update
  cadence, or whether bulky outputs (embeddings, long transcripts)
  should live in a sidecar index instead of inline on the Resource.
- **Links between notes.** Mentioning another note, quoting it,
  citing a source.
- **Location metadata** at the note level. Photos already carry EXIF
  location; whether we lift it to the note is an open question.
- **Spacenote graduation.** The relationship between a Nebulanote note
  promoted into a Spacenote note is not modeled yet.

## Open questions

- **Exact marker syntax.** `{{kind:<uuid>}}` is the working choice.
  Alternatives worth considering: `[[kind:<uuid>]]` (wiki style),
  short per-note ids (`{{image:p1}}`) for editable readability instead
  of raw UUIDs. Final call deferred.
- **Escape rule** for writing a literal `{{…}}` in `content`. Rare
  case; postpone until someone hits it.
- Should **content** support lightweight markup (markdown) within the
  text parts? Captures are often quick, and formatting discourages
  speed.
- Should **tags** be entity-level (a normalized set shared across the
  user's data, with rename/merge) or purely note-local strings that
  aggregate by convention?
- When a note is **edited**, do we keep **edit history** or only the
  latest state? Simpler: latest-only. Richer: a journal.
- Should a **note carry a location** (captured lat/lon) as a
  first-class optional field, given photos already have it in EXIF?
- How do we handle **very large Resources** (hours of video, scans
  of hundreds of pages)? Probably the same model, but worth naming as
  a stress case before committing to a blob strategy.
