# Nebulanote

A personal, single-user system for collecting text notes, voice memos,
photos, videos, and documents as a stream, with optional later
refinement by the user or an AI agent. The user owns the data, capture
is effortless, structure is added later (or never).

## Status

Very early. No code yet — only a vision document and data-model
sketches. Technology choices are deliberately deferred.

## Documentation

| File | Description |
|------|-------------|
| [`docs/vision.md`](docs/vision.md) | What Nebulanote is, core principles, scenarios, non-goals, open questions. Read first. |
| [`docs/data-model.md`](docs/data-model.md) | Logical data model — entities, fields, key decisions. |
| [`docs/storage.md`](docs/storage.md) | Physical storage — blob store, metadata store, per-platform data root. |

## Relationship to Spacenote

Separate from [Spacenote](https://github.com/mcbarinov/spacenote).
Spacenote is a structured, schema-first note system for small teams;
Nebulanote is a free-form personal inbox for one user. Optional
interop: a refined Nebulanote entry may be promoted into a Spacenote
note. See [`docs/vision.md`](docs/vision.md) for the full comparison.
