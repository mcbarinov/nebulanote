# Nebulanote Storage

> How Nebulanote data physically lives on disk. Complements
> [`data-model.md`](data-model.md), which describes the logical shape.

## Two stores, not one

Nebulanote splits persistence into two layers with very different
access patterns:

1. **Metadata store** — small structured records: Notes, Resources,
   Enrichments, indexes. Read often, written on edit. Needs
   queryability. Specific engine (SQLite, MongoDB, Postgres) is
   deferred to the tech-stack decision; the data model and this
   document do not depend on the choice.
2. **Blob store** — large immutable file bytes. Append-only.
   Content-addressed by SHA-256. Effectively a content-addressed
   filesystem.

Splitting them lets each be optimized for its own access pattern: the
metadata store can live in a database with indexes and transactions,
while the blob store stays as plain files on disk the user can
inspect, back up, and move with standard tools.

## Design principles

1. **User owns the bytes.** The blob store is a plain directory of
   files. No proprietary format, no database dependency, no
   cloud-only layer. `tar`, `rsync`, `du`, and a file manager all
   work directly on it.
2. **Same layout on every client.** Server, desktop, and mobile all
   use the same path scheme. Only the data root differs per platform
   (see below).
3. **Content-addressed and immutable.** A blob's path is derived from
   the SHA-256 of its bytes. Same bytes → same path, across time and
   devices. Blobs are never mutated in place.
4. **Atomicity via rename.** Writes always go through a temp location
   and `rename` into the final path, so no half-written blob is ever
   visible.
5. **Deduplication is free.** Content addressing means identical
   bytes collapse to one file automatically, on one device or across
   them.

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

- Hash the file bytes with SHA-256 → 64-character lowercase hex digest.
- First 2 hex chars = level-1 directory; next 2 hex chars = level-2
  directory; full 64-char hash = filename.
- Example: a blob with hash `abcd1234…ef01` lives at
  `blobs/sha256/ab/cd/abcd1234…ef01`.

Two-level sharding keeps any single directory manageable — at most
256 level-1 directories, 256 level-2 directories per level-1, and
blobs spread uniformly across them.

### No filename extensions

The hash uniquely identifies the bytes. The MIME type of the file is
recorded on the Resource record (see [`data-model.md`](data-model.md)).
Adding `.jpg` / `.mp4` to blob filenames would either duplicate that
information or lie about it (the same bytes are the same file no
matter what we call them).

### Atomic writes

Every upload follows the same dance:

1. Open a new file in `blobs/tmp/<upload-id>`.
2. Stream bytes into it, computing SHA-256 incrementally.
3. On completion, derive the final path from the hash.
4. If the final path already exists, discard the temp file (dedup hit).
5. Otherwise, `rename` the temp file to the final path. Under POSIX
   (and on modern Windows via equivalent APIs) this is atomic within
   the same filesystem.

This guarantees no consumer ever sees a partial blob at a real path.

### Idempotent upload

Adding a file whose hash already exists is effectively a no-op: the
temp file is discarded and the existing blob is reused. Uploading
the same photo a second time, from the same device or another
device, costs nothing beyond the hashing.

### Garbage collection

Run periodically, not synchronously on delete:

1. Enumerate all `blob_hash` values referenced by live Resources.
2. Walk `blobs/sha256/**` and identify files not in that set.
3. Delete orphans.

Details (quiet-window vs. two-phase mark-and-sweep, grace period
before deletion to tolerate in-flight writes) are deferred until we
actually implement GC.

### Portability and backup

Because the blob store is just a directory of files:

- `tar` / `rsync` handle backup and migration between machines.
- Filesystem-level encryption covers privacy at rest.
- `du` gives honest disk-usage accounting.
- A plain file manager lets the user inspect what they have.

No proprietary format, no application dependency on the blob side.
The user can always open their own data with standard tools.

## Per-platform data root

The layout is identical everywhere; only the root path differs,
following each OS's conventions.

| Platform | Data root |
|----------|-----------|
| Server (Linux) | Configured path, e.g. `/var/lib/nebulanote/` or a mounted volume |
| Desktop — macOS | `~/Library/Application Support/Nebulanote/` |
| Desktop — Linux | `$XDG_DATA_HOME/nebulanote/` (defaults to `~/.local/share/nebulanote/`) |
| Desktop — Windows | `%APPDATA%\Nebulanote\` |
| Android | App sandbox internal storage |
| iOS | App sandbox (Documents or Library directory) |

Platform-specific nuances that do **not** change the scheme:

- **Windows** has a historical 260-char path limit. Our full blob
  paths are ~70 chars; well within bounds.
- **iOS** may want the data directory tagged as excluded from iCloud
  auto-backup, depending on user choice.
- **Mobile** disk quotas are tighter than server/desktop, so
  cleanup/GC matters more in practice — but the mechanism is the
  same.

## Metadata store

Holds Note records, Resource records, Enrichment records, and any
derived indexes (e.g. "notes that reference a given Resource",
full-text index over Enrichment content).

Engine choice is deferred. Plausible candidates:

- **SQLite** — single-file, zero-config, a natural fit for
  local-first on mobile and desktop.
- **MongoDB** — document-oriented, matches the Resource + Enrichment
  shape well; what Spacenote uses.
- **Postgres** — strong relational and JSON support; fits a
  server-centric deployment.

The data model in [`data-model.md`](data-model.md) is engine-agnostic;
the same records fit into any of the above with minor adaptation.

## Sync hooks

Multi-device sync is its own design topic, not specified here. What
this storage layout gives sync for free:

- **Blob sync is trivially content-addressable.** Devices exchange
  hash lists; missing blobs flow in either direction; conflicts are
  impossible because identical hashes mean identical bytes.
- **Metadata sync is the interesting part.** Records carry UUIDv7
  ids, `updated_at`, and `deleted_at` as hooks for last-writer-wins
  merge or a richer protocol. Designed in a separate doc when we
  reach that work.
