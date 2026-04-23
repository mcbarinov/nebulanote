# Nebulanote Data Model

> Draft. Describes the **logical** data model. Storage engine, sync
> protocol, indexing, and persistence format are deferred to other
> docs. Models below are pydantic skeletons — types and inline
> comments only, no validators or methods.

## Design principles

1. **Multi-device by birth.** Every record gets a **UUIDv7** at
   creation on any device — globally unique and sortable by
   creation time.
2. **Two kinds of records.** A **Note** is what the user wrote; a
   **Resource** is a file-shaped thing they captured (photo, voice
   memo, video, document). Notes reference Resources by id;
   Resources live on their own and can be referenced by many Notes.
3. **Large content lives outside records in a content-addressed
   blob store.** Notes and Resources are small structured records.
   File bytes live separately, keyed by SHA-256. Same bytes → same
   id, so dedup is free and sync can skip blobs another device
   already has.
4. **Soft delete.** Deletion is marked, not erased, so it
   propagates to other devices during merge. GC of tombstones and
   orphaned blobs is a separate later concern.
5. **Minimal required fields.** Capture must be effortless; the
   minimum for a valid note is small.

## Key decisions

- **A Note's content is a single string with inline Resource refs.**
  One `content` field. Resources are referenced inline via markers of
  the form `{{kind:<resource-uuid>}}`. Ordering and interleaving are
  expressed by position in the string — no separate attachments list.
- **The Note does not carry an attachments list.** The set of
  referenced Resources is derived by parsing `content`. Reverse-lookup
  indexes are a storage concern, not a data-model field.
- **Content must be non-empty.** A voice-only Note is valid — its
  `content` is just the single marker.
- **Capture creates both a Resource and a Note.** Taking a photo
  produces a Resource and a minimal Note whose `content` is just the
  marker. Keeps the stream complete and avoids orphans at capture.
- **Every inline ref must resolve** to a live Resource; the marker
  `kind` must match the Resource's `kind`.
- **Resource is minimal; kind-specific data splits by lifecycle.**
  Resource carries only universal fields plus an embedded `capture`
  block (1:1, typed, from the file itself). AI-derived data — OCR,
  transcripts, descriptions, embeddings — lives in separate
  **Enrichment** records (0..N, joined by `resource_id`). Capture
  embedded because it's tiny, always needed, immutable. Enrichments
  separate because they're bulky, optional, append-only, unbounded.
- **Resources should have at least one referencing live Note.**
  When the last referencing Note is deleted, the Resource becomes an
  **orphan candidate** — flagged for user review, not auto-deleted.
- **Comments are human-authored text annotations on Notes.** A
  separate entity (not a Note field, not a `content` edit) so
  original capture and later reflections keep their own timestamps.
  Same marker syntax as Note content. Flat, not threaded. Machine
  annotations are a separate future entity.
- **Tags on Notes are plain lowercase strings.** No hierarchy,
  free-form. Added by user or AI.
- **Provenance lives in a `meta: dict[str, Any]` bag.** Both `Note`
  and `Resource` carry `meta`, scoped strictly to provenance (where
  the record came from, what produced it). Not a general dumping
  ground — content, annotations, tags, and AI outputs have their own
  homes. Conventional keys documented in the Provenance section.

## Core entities

### Note

The user's expression — a `content` string that interleaves plain
text with inline Resource references.

```python
class Note(BaseModel):
    id: UUID  # UUIDv7 — sortable by creation time, globally unique
    created_at: datetime  # when the note was captured, on the source device
    updated_at: datetime  # last edit time; equals created_at if never edited
    deleted_at: datetime | None  # soft-delete tombstone; None means live

    content: str  # free-form text with inline refs like {{image:<resource-uuid>}}
    tags: list[str]  # lowercase tag strings, unique within the note

    meta: dict[str, Any]  # provenance-only bag; see Provenance metadata below
```

Stream order is by `created_at` (equivalent to ordering by `id` under
UUIDv7). `updated_at` and `deleted_at` are the hooks a future sync
mechanism uses for merge and deletion propagation.

### Content and inline references

`content` is the single source of truth for what the note says, what
Resources it uses, and in what order.

**Marker syntax:** `{{kind:<resource-uuid>}}`, where `kind` is one of
`image`, `audio`, `video`, `document` and matches the referenced
Resource's `kind`. Examples:

```
Trying a new recipe today {{image:3f1a…}} — smelled great.
{{audio:9b2c…}}
Test results from today {{document:1d4e…}} will chat with the doctor tomorrow.
```

**Invariants:** every marker must resolve to a live Resource; marker
`kind` must match the Resource's `kind`. Escape rule for literal
`{{…}}` in text is deferred.

### Comment

Human-authored text annotation on a Note. Separate entity so the
original capture and later reflections keep their own timestamps
(rather than appending to `Note.content`).

```python
class Comment(BaseModel):
    id: UUID  # UUIDv7
    note_id: UUID  # FK → Note.id
    created_at: datetime
    updated_at: datetime  # last edit; equals created_at if never edited
    deleted_at: datetime | None  # soft-delete tombstone
    content: str  # same {{kind:<id>}} marker syntax as Note.content
    meta: dict[str, Any]  # provenance bag
```

Human-authored only (no `author` field). Flat, not threaded.
Machine-produced annotations are a separate future entity.

### Resource

A file-shaped thing in the archive: photo, voice memo, video,
document. Has identity independent of any Note — the same Resource
can be referenced from many Notes.

```python
class Resource(BaseModel):
    id: UUID  # UUIDv7 — used in note markers like {{image:<id>}}
    created_at: datetime  # when the resource was added to the system
    updated_at: datetime  # last time the core record was touched
    deleted_at: datetime | None  # soft-delete tombstone

    kind: Literal["image", "audio", "video", "document"]  # coarse class
    mime: str  # authoritative MIME type, e.g. "image/jpeg"
    blob_hash: str  # SHA-256 hex of the file bytes; key into the blob store
    size_bytes: int  # file size in bytes

    capture: ImageCapture | AudioCapture | VideoCapture | DocumentCapture | None
    # ^ kind-specific metadata from the file; None if unparsable

    meta: dict[str, Any]  # provenance bag
```

### Per-kind capture records

Embedded on Resource via `capture`, one type per `kind`. Holds only
what the file reveals at upload — synchronous, no AI, no network.

```python
class ImageCapture(BaseModel):
    width: int  # pixels
    height: int  # pixels
    exif: dict[str, Any] | None  # raw EXIF dict if the file carries it

class AudioCapture(BaseModel):
    duration_ms: int  # length in milliseconds
    codec: str | None  # short codec label (e.g. "aac", "opus")

class VideoCapture(BaseModel):
    duration_ms: int  # length in milliseconds
    width: int  # pixels
    height: int  # pixels
    codec: str | None  # short codec label (e.g. "h264", "av1")

class DocumentCapture(BaseModel):
    filename: str | None  # original filename from upload, if known
    page_count: int | None  # for paginated documents (e.g. PDF)
```

If parsing fails, `capture` is `None` and the UI falls back to
"unknown". The Resource is still valid (blob, size, mime exist) —
capture is best-effort, not an invariant. `ImageCapture.exif` stays
`dict[str, Any]` because EXIF keys are camera-defined and not worth
pre-typing.

### Enrichment

Data derived about a Resource *after* capture — typically by AI.
Separate collection, one record per derived item, joined by
`resource_id`.

```python
class Enrichment(BaseModel):
    id: UUID  # UUIDv7
    resource_id: UUID  # FK to Resource.id
    created_at: datetime  # when this enrichment was produced
    kind: str  # "transcript", "ocr", "description", "tags", "embedding", "summary", ...
    generator: str  # "whisper-large-v3", "gpt-4o-2025-xx", "manual", "tesseract-5", ...
    content: dict[str, Any]  # kind-specific payload; shape varies by `kind`
```

`content` is `dict[str, Any]` because the set of enrichment kinds
will grow and each has its own natural shape (transcript has `text`
+ `language`; embedding has `vector` + `dim`; OCR may carry per-page
text). Specific kinds can be promoted to typed sub-models once they
stabilize.

**Why Enrichments sit beside Resource, not inside it:**

- The stream view renders many Resources as thumbnails and must not
  pull heavy enrichments (embedding vectors are ~6 KB each).
- Each Enrichment syncs across devices as its own record; adding a
  new AI run is an INSERT, not an array mutation.
- Full-text and vector indexes live naturally on a dedicated
  collection.

To render "everything about a Resource" the service layer runs two
queries and composes a view object in memory — not persisted, just a
convenience.

### Capture and resource lifecycle

- **Capture** creates a Resource and a minimal Note whose `content`
  is just the marker. Stream always contains every captured thing.
- **Re-reference.** A later Note can include `{{image:<old-id>}}`
  to weave in a previously captured Resource. No copy is made.
- **Note delete** does not cascade to Resources. A Resource whose
  every referencing Note is deleted becomes an orphan candidate —
  surfaced for user review, not auto-deleted.
- **Timestamps are distinct.** `Resource.created_at` is when the
  file entered the system; `Note.created_at` is when the writing
  happened. UIs must not conflate them.

### Provenance metadata (`meta`)

`Note` and `Resource` (and `Comment`) each carry `meta: dict[str, Any]`
— a flat bag of optional context about *where the record came from*.
Nothing else goes here: content, tags, annotations, and AI outputs all
have their own homes.

If a field is tempting to put in `meta` but describes *what the record
is* rather than *where it came from*, it belongs elsewhere.

**Conventional keys** (by convention, not typed; absent by default;
unknown keys ignored so old clients can read newer records):

| Key | When present | Example |
|---|---|---|
| `source_device` | Captured on a client device. | `"pixel-8"`, `"macbook-m"` |
| `generator` | Produced by an agent, not a human. | `"health-aggregator-v1"` |
| `imported_from` | Came from a bulk import. | `"apple-notes-export-2026-01"` |
| `client_version` | (future) Client build, for bug reports. | `"1.4.2"` |
| `location` | (future) GPS at capture, if user opts in. | `{"lat": 55.7, "lon": 37.6}` |

Static typing over `meta` keys is given up deliberately — provenance
is used for UI decoration, filtering, and debugging, not
correctness-critical paths. Hot keys can be promoted to typed fields
later if needed.

### Blob (storage concept, not a modeled record)

File bytes addressed by SHA-256. Not a pydantic model — lives in a
content-addressed blob store on the filesystem, referenced via
`Resource.blob_hash`. Physical layout is in [`storage.md`](storage.md).

- Same bytes → same blob, so dedup is automatic across Resources.
- Deleting a Resource does not delete the blob. Blob GC runs
  separately when no live Resource references it.

## What we are not modeling yet

Deferred; listed so nothing goes missing.

- **Sync metadata.** Device IDs, vector clocks, merge journals. The
  model leaves hooks (UUIDs, `updated_at`, `deleted_at`) but does
  not specify a mechanism.
- **Groups / curated notes.** Collapsing raw entries into a refined
  summary. Candidates: a Note that references other Notes, or a
  dedicated `Group` entity.
- **Sharing state.** Public / password-protected visibility of
  individual notes. Needs its own record.
- **Enrichment content schemas.** Per-kind payloads (what a
  `transcript` contains, what an `embedding` looks like) are
  deferred until AI pipelines run; committing shapes now would be
  guessing.
- **Machine-authored annotations on Notes.** When AI agents attach
  structured output to a Note (aggregations, trend analysis,
  extracted facts), that goes in a future `Annotation` entity,
  probably shaped like:

  ```python
  class Annotation(BaseModel):
      id: UUID
      note_id: UUID                   # FK → Note.id
      created_at: datetime
      deleted_at: datetime | None
      kind: str                       # "health_summary", "aggregation", ...
      generator: str                  # agent id, typed because identity-level
      content: dict[str, Any]         # structured payload; shape per kind
      text: str | None                # optional human-readable preview
      meta: dict[str, Any]            # provenance bag
  ```

  Named here so the `Comment` (human) vs. `Annotation` (machine)
  split is anticipated when the time comes.

- **Agents as registered entities.** Agents that produce
  `Annotation`s or auto-create Notes will need a registry (id, task,
  triggers, scope, permissions). The model reserves the signing hooks
  (`meta["generator"]`; typed `generator` on future `Annotation`);
  the registry itself is a later concern.

- **Links between notes.** Mentioning another note, quoting it,
  citing a source.
- **Location metadata** at the note level. Photos already carry EXIF
  location; whether we lift it to the note is an open question.
- **Spacenote graduation.** The relationship between a Nebulanote note
  promoted into a Spacenote note is not modeled yet.

## Open questions

- **Marker syntax.** `{{kind:<uuid>}}` is the working choice.
  Alternatives: `[[kind:<uuid>]]` (wiki), short per-note ids
  (`{{image:p1}}`) for readability.
- **Enrichment cardinality per `(resource_id, kind)`.** Many (keep
  old transcripts for comparison) or latest-wins? Shape supports
  both.
- **Fallback `kind`** for files outside image/audio/video/document
  (spreadsheets, archives, unknown binaries). Catch-all `"file"`
  kind or reject at upload?
- **Escape rule** for literal `{{…}}` in `content`. Postpone.
- **Markdown** in `content` — allow, or stay plain? Formatting can
  slow down capture.
- **Tags** — note-local strings (simple) or entity-level with
  rename/merge (richer)?
- **Edit history** on Notes — latest-only (simple) or full journal?
- **Location** (lat/lon) at the Note level — promote to first-class
  field, or leave it to EXIF on photos?
- **Very large Resources** (hours of video, hundred-page scans) —
  same model or a chunking scheme?
