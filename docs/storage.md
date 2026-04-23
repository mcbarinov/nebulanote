# Nebulanote Storage

> How Nebulanote data physically lives on disk. Complements
> [`data-model.md`](data-model.md), which describes the logical shape.

## Two stores

1. **Metadata store** — small structured records (Notes, Resources,
   Enrichments, Comments, indexes). Read often, needs queryability.
   Engine (SQLite / MongoDB / Postgres) is deferred.
2. **Blob store** — large immutable file bytes, content-addressed by
   SHA-256. Plain files on disk, the same layout on every client.

Splitting them lets each be optimized for its access pattern and
keeps the blob side as a plain directory the user can inspect, back
up, and move with standard tools.

## Blob store layout

```
<data-root>/
├── db/                    # metadata store (engine-specific files)
└── blobs/
    ├── tmp/               # staging area for in-progress uploads
    └── sha256/
        └── ab/cd/abcd…    # one blob per file; path = hash[:2]/hash[2:4]/full-hash
```

### Path scheme

SHA-256 the bytes → 64-char lowercase hex. Path =
`blobs/sha256/<hash[:2]>/<hash[2:4]>/<full-hash>`. Example:
`abcd1234…ef01` → `blobs/sha256/ab/cd/abcd1234…ef01`.

Two-level sharding spreads blobs uniformly, avoiding the
"millions of files in one directory" pathology.

No filename extension. The hash identifies bytes; MIME lives on the
Resource. Adding `.jpg`/`.mp4` to a hash-named file would duplicate
that info or lie about it.

### Atomic writes

1. Stream bytes into `blobs/tmp/<upload-id>`, hashing as you go.
2. On completion, derive the final path.
3. If it already exists → discard temp (dedup hit).
4. Otherwise `rename` temp into place. POSIX rename is atomic within
   a filesystem, so consumers never see a partial blob.

### Idempotent upload

If the target hash already exists, the temp file is discarded and
the existing blob is reused. Re-uploading the same bytes costs only
the hashing.

### Garbage collection

Run periodically, not synchronously on delete:

1. Enumerate `blob_hash` values on live Resources.
2. Delete any file in `blobs/sha256/**` not in that set.

Concurrency details (quiet window vs. two-phase mark-and-sweep) are
deferred until GC is actually implemented.

## Per-platform data root

Same layout everywhere; only the root path differs.

| Platform | Data root |
|----------|-----------|
| Server (Linux) | Configured path, e.g. `/var/lib/nebulanote/` |
| Desktop — macOS | `~/Library/Application Support/Nebulanote/` |
| Desktop — Linux | `$XDG_DATA_HOME/nebulanote/` |
| Desktop — Windows | `%APPDATA%\Nebulanote\` |
| Android | App sandbox internal storage |
| iOS | App sandbox (Documents or Library) |

Minor nuances that don't change the scheme: Windows' 260-char path
limit (our paths are ~70 chars, fine); on iOS the directory may want
to be excluded from iCloud auto-backup; mobile disk quotas are
tighter so GC matters more in practice.

## Metadata store

Holds Note, Resource, Comment, Enrichment records plus any derived
indexes (reverse-lookup, full-text, vectors). Engine deferred.
Candidates:

- **SQLite** — single-file, zero-config, natural for local-first
  mobile and desktop.
- **MongoDB** — document-oriented, matches Resource + Enrichment
  shape; what Spacenote uses.
- **Postgres** — strong relational and JSON; fits server-centric
  deployment.

The data model in [`data-model.md`](data-model.md) is engine-agnostic.

## Sync hooks

Sync protocol is its own doc. What this layout gives it for free:

- **Blobs** are trivially sync-friendly: devices exchange hash
  lists; missing blobs flow; identical hashes ≡ identical bytes, so
  no conflicts.
- **Metadata** records carry UUIDv7 ids, `updated_at`, and
  `deleted_at` as hooks for last-writer-wins or a richer protocol.
