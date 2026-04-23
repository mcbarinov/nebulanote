# Nebulanote Vision

Nebulanote is a personal, single-user system for collecting
everything one person wants to keep, as a stream of entries from
any device, with the lowest possible friction at capture. The
data belongs to the user, not a cloud service. Structure is added
later, on demand, if at all.

## Primary scenarios

- **Health log.** Symptoms, medications, test results, photos of
  prescriptions. At a doctor's appointment, the user asks an AI
  assistant to surface relevant history on the spot.
- **Daily activity inbox.** Voice notes, photos, and short texts
  about workouts, meals, ideas throughout the day. Curation
  collapses related fragments into tidy summaries later.
- **Personal archive.** Scanned documents, receipts, screenshots,
  links, fleeting thoughts — in one place the user controls.

## What goes into the stream

- Text notes.
- Voice memos.
- Photos.
- Videos.
- Documents (PDFs, scans, etc.).
- Wearable health data — heart rate, sleep, steps and similar
  samples from a watch, band, or ring.
- Location history — GPS tracks or visited-places records.
  Ingest shape is an open question.

## Core principles

1. **The user owns the data.** No dependency on any external
   cloud service. Data must be exportable in formats that
   survive the application.
2. **Capture comes first.** Adding content must be faster and
   simpler than anywhere else. No schema, no fields, no mandatory
   tags.
3. **All data kinds are first-class.** None is privileged over
   the others.
4. **Multi-device capture is essential.** Phone, web, desktop —
   content captured anywhere merges into the same stream.
5. **Every client is optional.** A user who only wants the mobile
   app should be able to run Nebulanote as a self-contained
   mobile app.
6. **AI is optional but central to the design.** The system is
   built so an AI agent can read, search, organize, and answer
   questions over the stream. The user picks local, cloud, or
   none.

## Stream-to-curation lifecycle

Content enters as a stream of raw entries — nothing else required
at capture time. Later, manually or with AI help, entries can be:

- **Grouped** — ten fragments about today's workout collapsed
  into one summary entry.
- **Refined** — a rough voice memo turned into a clean note.
- **Deleted** once folded into something richer.

Capture is cheap. Curation is separate, later, and entirely
optional.

## Sharing

Private by default. Individual entries can be opted into sharing
(public post, password-protected link) without exposing the rest
of the stream.

## Non-goals

- Multi-user collaboration or shared workspaces.
- Schema-first data entry.
- Being a cloud-only service the user cannot run independently.

## Relationship to Spacenote

Nebulanote is independent from
[Spacenote](https://github.com/mcbarinov/spacenote). Spacenote
is structure-first and built for small teams; Nebulanote is a
free-form personal stream for one user. Spacenote is a filing
cabinet, Nebulanote is an inbox. Optional interop: a refined
Nebulanote entry can be promoted into a Spacenote note.
Promotion semantics (one-way export, linked copy, two-way sync)
are not decided.

## Open questions

- **Data format** for portability and long-term survival.
- **Sync topology** across devices (direct, via server, via
  user-owned file sync, or a mix).
- **AI model selection** and the privacy boundary for cloud
  models.
- **Sharing mechanics**: hosting, URL format, expiration,
  passwords.
- **Location ingest** — source, format, frequency.
