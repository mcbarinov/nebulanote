# Nebulanote Vision

Nebulanote is a personal, single-user system for collecting everything
one person wants to keep — text notes, voice memos, photos, videos,
documents — as a stream of entries from any device, with the lowest
possible friction at capture. The data belongs to the user, not a
cloud service. Structure and meaning are added later, on demand, if
at all.

## How it differs from Spacenote

- Spacenote: structure first, content fits the structure.
- Nebulanote: content first, structure emerges later (or never).
- Spacenote: 1–10 users, shared spaces, permissions.
- Nebulanote: one user, optional sharing of selected items.
- Spacenote is a filing cabinet. Nebulanote is an inbox.

The two are independent. Optional interop: a refined Nebulanote entry
may be promoted into a Spacenote note.

## Core principles

1. **The user owns the data.** No dependency on any external cloud
   service. Data must be exportable in formats that survive the
   application.
2. **Capture comes first.** Adding content must be faster and simpler
   than anywhere else. No schema, no fields, no mandatory tags.
3. **Any content type is first-class.** Text, voice, photo, video,
   document — none is privileged over the others.
4. **Multi-device capture is essential.** Phone, another phone, web,
   desktop, and in the future new surfaces (smart speakers, …).
   Content captured anywhere merges into the same stream.
5. **Every client is optional.** A user who only wants the mobile app
   should be able to run Nebulanote as a self-contained mobile app.
6. **AI is optional but central to the design.** The system is built
   so an AI agent can read, search, organize, and answer questions
   over the stream. The user picks local, cloud, or none.
7. **Optional Spacenote integration.** When connected, refined
   entries can graduate into Spacenote notes.

## Stream-to-curation lifecycle

Content enters as a stream of raw entries — nothing else required at
capture time. Later, manually or with AI help, entries can be:

- **Grouped** — ten fragments about today's workout collapsed into
  one summary entry.
- **Refined** — a rough voice memo turned into a clean note.
- **Promoted** into a Spacenote note when structure is wanted.
- **Deleted** once folded into something richer.

Capture is cheap. Curation is separate, later, and entirely optional.

## Primary scenarios

- **Health log.** Symptoms, medications, test results, photos of
  prescriptions. At a doctor's appointment, the user asks an AI
  assistant to surface relevant history on the spot.
- **Daily activity inbox.** Voice notes, photos, and short texts
  about workouts, meals, ideas throughout the day. Curation
  collapses related fragments into tidy summaries later.
- **Personal archive.** Scanned documents, receipts, screenshots,
  links, fleeting thoughts — in one place the user controls.

## Sharing

Private by default. Individual entries can be opted into sharing
(public post, password-protected link) without exposing the rest of
the stream.

## Non-goals

- Multi-user collaboration or shared workspaces.
- Schema-first data entry.
- Replacing Spacenote or duplicating its structured-notes role.
- Being a cloud-only service the user cannot run independently.

## Open questions

- **Data format** for portability and long-term survival.
- **Sync topology** across devices (direct, via server, via
  user-owned file sync, or a mix).
- **AI model selection** and the privacy boundary for cloud models.
- **Graduation semantics** to Spacenote: one-way export, linked copy,
  or two-way sync.
- **Sharing mechanics**: hosting, URL format, expiration, passwords.
