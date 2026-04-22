# Nebulanote Vision

Nebulanote is a personal, single-user system for capturing everything one
person wants to keep — text notes, voice memos, photos, videos, documents —
as a simple stream of entries, with optional later refinement by the user
or an AI agent.

## What nebulanote is

A single place where one person collects all their own content, from any
device they use, with the lowest possible friction at the moment of
capture. The data belongs to the user, not to a cloud service. Structure
and meaning are added later, on demand — never as a precondition for
writing something down.

## How it differs from spacenote

Spacenote is a structured, schema-first note system for small teams. Every
note lives in a space, follows a set of fields, and has a predefined
shape. Nebulanote is the opposite: a free-form personal stream where
anything can be added instantly without choosing a schema, a space, or
even a type.

- Spacenote: structure first, content fits the structure.
- Nebulanote: content first, structure emerges later (or never).
- Spacenote: 1–10 users, shared spaces, permissions.
- Nebulanote: one user, optional sharing of selected items.
- Spacenote is a filing cabinet. Nebulanote is an inbox.

The two systems are independent and each works on its own. They can
optionally interoperate: a refined nebulanote entry may be promoted into
a structured spacenote note.

## Core principles

1. **The user owns the data.** No dependency on any external cloud
   service. Data must be stored and exportable in formats that survive
   the application.
2. **Capture comes first.** Adding content must be faster and simpler
   than anywhere else. No schema, no fields, no mandatory tags.
3. **Any content type is first-class.** Text, voice, photo, video,
   document attachments — none is privileged over the others.
4. **Multi-device capture is essential.** A user records from whichever
   surface is at hand: a phone, another phone, the web, a desktop, and
   in the future new surfaces such as smart speakers. Content captured
   anywhere merges into the same personal stream.
5. **Every client is optional.** Someone who only wants the mobile app
   should be able to use nebulanote as a self-contained mobile app. No
   single client is required for the system to work.
6. **AI is optional but central to the design.** The system is built so
   an AI agent can read, search, organize, and answer questions over the
   stream. The user chooses whether to use a local model, a cloud model,
   or no AI at all.
7. **Optional spacenote integration.** Nebulanote works perfectly well
   without spacenote. When connected, refined entries can graduate into
   spacenote notes.

## The stream-to-curation lifecycle

Content enters as a stream of raw entries. Nothing else is required of
the user at capture time.

Later, manually or with AI help, entries can be:

- **Grouped** — e.g. ten fragments about today's workout collapsed into
  one summary entry that lists them.
- **Refined** — e.g. a rough voice memo turned into a clean note.
- **Promoted** into a spacenote note, when structure is wanted.
- **Deleted**, once their content has been folded into something richer.

Raw capture is cheap. Curation is a separate, later, and entirely
optional act.

## Primary scenarios

- **Health log.** Symptoms, medications, test results, photos of
  prescriptions collected over time. At a doctor's appointment, the user
  asks an AI assistant by voice to surface relevant history on the spot.
- **Daily activity inbox.** Throughout the day the user dumps voice
  notes, photos, and short texts about workouts, meals, ideas. Later,
  curation collapses related fragments into tidy summaries.
- **Personal archive.** Anything worth keeping — scanned documents,
  receipts, screenshots, links, fleeting thoughts — in one place the
  user controls.

## Sharing

Content is private by default. Any individual entry can be opted into
sharing, for example as a public post or a password-protected link,
without exposing the rest of the stream.

## Non-goals

- Multi-user collaboration or shared workspaces.
- Schema-first data entry.
- Replacing spacenote or duplicating its structured-notes role.
- Being a cloud-only service the user cannot run independently.

## Open questions

These are deliberately deferred. The vision does not depend on a
specific answer to any of them.

- **Data format** for portability and long-term survival.
- **Sync topology** across devices (direct, via a server, via a
  user-owned file sync layer, or some combination).
- **Entry atomicity**: is one entry a single media item, or a message
  with text plus multiple attachments?
- **AI model selection** and the privacy boundary for cloud models.
- **Graduation semantics** to spacenote: one-way export, linked copy,
  or two-way sync.
- **Sharing mechanics**: hosting, URL format, expiration, passwords.
