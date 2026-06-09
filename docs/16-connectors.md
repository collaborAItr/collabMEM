# 16. Connectors and Source Channels

The architecture so far assumes one input mode: the user types into a chat box, an assistant responds, and extraction runs over the resulting `(userMessage, modelResponse)` pair. That covers the dominant case but it's narrow. Most people working with an assistant for any non-trivial period also throw at it:

- Files (PDFs, transcripts, design docs, exports from other tools).
- Calendar events ("here's what's on my plate this week").
- Email or message threads ("read this and remember the gist").
- Captured content from a browser ("this article").

Connectors are the seam that lets all of these flow into the same extraction pipeline without each one growing its own bespoke ingest stack.

---

## What a connector is

A **Connector** is a small adapter that takes a non-chat input, normalizes it into the same `(userMessage, modelResponse)` shape the extractor already understands, and runs the resulting synthetic turn through the standard pipeline.

Concretely: a file-upload connector receives `{ filename, contents }`, frames it as `userMessage = "Uploaded file: <filename>"` and `modelResponse = <extracted plaintext>`, and hands the pair to the extractor. The extractor doesn't know or care that there was no live model call — it sees the same shape it always sees.

The result is that every memory pathway already built for chat (extraction, reconciliation, embedding, sync, inspector surfaces) works for files, calendar events, and anything else the moment a connector exists for it.

---

## The source-channel contract

Every memory created through a connector carries a **`source_channel`** field stamped by the connector:

```typescript
type SourceChannel = "chat" | "file_upload" | "calendar" | "email" | string;
```

`source_channel` is plumbing the user can see. It powers:

- **Provenance.** "Where did this engram come from?" must answer with more than "an extraction" — it must say which channel.
- **Filters.** The Memory Inspector should let the user scope the engram view by channel ("show me only memories from uploaded files").
- **Routing decisions.** Some downstream surfaces (e.g. a "recent uploads" dashboard) only make sense for non-chat channels.
- **Quota and throttling.** A runaway file-import script shouldn't be able to drown the chat-stream extractor; per-channel rate limits live above the connector and read this field.

`source_channel` defaults to `"chat"` for legacy rows, so the column is safe to add to an existing memory store without backfill.

---

## What a connector is *not*

- **Not a custom extractor.** A connector adapts an *input* into the standard shape; it doesn't replace the extraction pipeline. If a channel needs a fundamentally different extraction policy, that's a fork in the extractor, not a connector concern.
- **Not a transport layer.** Authentication, rate limiting, file size limits, and binary-to-text decoding (PDF parsing, OCR, etc.) live above the connector. The connector receives normalized text.
- **Not a memory bypass.** Everything a connector ingests goes through the same reconciliation and inspector surfaces as chat-derived memory. There is no "trusted import" path.
- **Not aware of channel semantics it doesn't need.** A file-upload connector doesn't need to know what a calendar event is. Each connector is small and focused on one thing.

---

## Registry pattern

Connectors register themselves with a central registry at startup so the ingest endpoint can dispatch by channel name without knowing about each one statically:

```
POST /connectors/:channel/ingest  →  registry.get(channel).ingest(payload, ctx)
```

The registry is a flat map; there is no inheritance hierarchy beyond the abstract base. New connectors are pure additions — they ship without modifying any other connector or any pipeline code.

---

## Synthetic conversation ids

Every connector ingest call needs to attach to *some* conversation so the extractor's per-conversation logic (turn numbering, retry queues, salient digest stream) keeps working. Use a synthetic id of the shape:

```
connector:<channel>:<userId>
```

Per-user, per-channel buckets are usually right: a user's uploaded files form one rolling synthetic conversation, their calendar another. The chat surface filters synthetic conversations out of the conversation list so they don't pollute the user's chat history; the Memory Inspector still shows the engrams they produced.

---

## Per-channel boundaries

- **File upload** is the canonical reference connector. It accepts `{ filename, contents }`, caps payload size at a sensible bound (say 200 KB of text per call — chunk above that on the client), and frames the payload as the synthetic model response. Larger files post in multiple calls; the extractor's normal batching handles the resulting turn stream.
- **Calendar** connectors should normalize each event into one synthetic turn whose `userMessage` is "Calendar event: <title>" and whose `modelResponse` is the event description. The temporal information becomes a salient digest, just like in chat.
- **Email / thread** connectors should ingest one message per turn rather than the whole thread as a single blob — the extractor treats turns as units, and a bundled thread becomes one massive engram instead of one per author intent.

The pattern generalizes: pick the smallest atomic unit of the source format and turn each one into a synthetic turn.

---

## What to read next

- [04-asset-taxonomy.md](04-asset-taxonomy.md) — the assets that connector-driven extractions ultimately produce
- [06-extraction.md](06-extraction.md) — the pipeline that connector payloads flow through
- [10-transparency-mutability.md](10-transparency-mutability.md) — how `source_channel` should appear in inspector filters and provenance
