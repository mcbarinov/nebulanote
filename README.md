# Nebulanote

A personal, single-user system for capturing everything one person wants
to keep — text notes, voice memos, photos, videos, documents — as a
simple stream of entries, with optional later refinement by the user or
an AI agent.

The guiding idea: the user owns the data, capture must be effortless,
and structure is added later (or never), not up front.

## Status

Very early. No code yet. The project exists as a vision document while
scope and principles are being nailed down. Technology choices are
deliberately deferred.

## Documentation

| File | Description |
|------|-------------|
| [`docs/vision.md`](docs/vision.md) | What Nebulanote is, core principles, scenarios, non-goals, and parked open questions. Read first. |
| [`docs/data-model.md`](docs/data-model.md) | Logical data model — entities, fields, key decisions. Draft. |
| [`docs/storage.md`](docs/storage.md) | Physical storage layout — blob store on disk, metadata store, per-platform data root. Draft. |

## Relationship to Spacenote

Nebulanote is a separate project from [Spacenote](https://github.com/mcbarinov/spacenote). The two
are independent and each works on its own. Spacenote is a structured,
schema-first note system for small teams. Nebulanote is a free-form
personal inbox for one user. They can optionally interoperate: a refined
Nebulanote entry may be promoted into a structured Spacenote note.

See `docs/vision.md` for the full comparison.
