# Setup guide

[← Back to README](README.md)

This is the full wiring guide. If you haven't read the [README](README.md) yet, start there for the project pitch and quickstart. This document walks through everything you need to take `index.html` from "works in localStorage only" to a live, hourly-refreshing dashboard wired into your tools.

## What you're working with

- **`index.html`** — single-file dashboard. Click around to confirm closures, notes, and attachments work via `localStorage`. They do.
- **This guide** — wiring instructions for the four integration points.

## What it shows

Top fold:

- **Next 24h key meetings** — checkable, with optional notes and doc attachments per line.
- **Cross-program clashes** — auto-derived conflicts the synthesis surfaced (resource collisions, sequencing dependencies, schedule clashes).

Per program card:

- **Status dot** — green / yellow / red rolled up from the RAID.
- **Next 3 for you** — your top three actions on this program right now.
- **Decisions on others** — people whose answers you're waiting on, with age.
- **Open RAID** — Risks / Assumptions / Issues / Dependencies. Hover any item → popover with the source-doc refs for that program.
- **What's hot** — a paragraph of synthesis prose tying together the freshest signals.
- **Refs** — collapsible list of the source docs the synthesis pulled from.

Toolbar:

- **💬 Ask** (Cmd/Ctrl+K) — drawer that snapshots the live dashboard (closures, notes, attachments, RAID) and sends it as grounding context to an AI of your choice.
- **↺ Reset closures** — un-checks everything you've closed.
- **⟳ Refresh** — triggers your scheduled refresh task on demand.

## How it works (architecture)

```
┌──────────────────┐    ┌─────────────────┐    ┌──────────────┐
│  Signal sources  │───▶│  Refresh task   │───▶│ index.html   │
│ (Gmail, Drive,   │    │ (cron, daily or │    │ (regenerated │
│  Calendar, Slack,│    │  hourly — runs  │    │  in place)   │
│  AI search,      │    │  via AI agent + │    │              │
│  task tracker,   │    │  tool-using AI) │    │              │
│  notes-system)   │    │                 │    │              │
└──────────────────┘    └────────┬────────┘    └──────┬───────┘
                                 │                    │
                                 │ ← reads state.md   │ closures / notes
                                 │   for context      │ / attachments
                                 │                    ▼
                          ┌──────┴─────────────────────────┐
                          │      Cloud storage             │
                          │  [CTRL-TWR-STATE].md           │
                          │  [CTRL-TWR-EDITS].log          │
                          └────────────────────────────────┘
```

Key idea: the dashboard is **regenerated**, but your state (closures, notes, attachments) is **preserved** across regenerations because it lives in cloud storage, not in the HTML. Each regeneration seeds itself from the storage snapshot on load.

## Wiring it up

The script in `index.html` has four `WIRE:` markers. Search for them and replace the stubs with calls to whatever tooling you have.

### `WIRE: REFRESH` — your refresh task

The refresh task is where the magic happens. It's a scheduled job that:

1. **Pulls fresh signal** from your tools — calendar events, recent email threads, doc edits, chat messages mentioning your programs, AI search summaries, task tracker activity. Use parallel calls; the synthesis quality is bottlenecked on recency, not volume.
2. **Reads `[CTRL-TWR-STATE].md`** from cloud storage to know what you've already closed and what notes you've added. Closed items don't resurface unless something materially new happens.
3. **Feeds notes as user-provided context** to the AI synthesis prompt — your inline corrections override the raw source data.
4. **Synthesizes per-program** with an AI model: status dot, Next 3, Decisions on others, RAID, What's hot, Refs.
5. **Auto-derives cross-program clashes** by comparing time conflicts and shared stakeholders across the per-program syntheses.
6. **Regenerates `index.html`** with the new content, preserving CSS and JS.

Recommended cadence: hourly on workdays during work hours (e.g. cron `0 8-19 * * 1-5`). The dashboard shows "last refresh" in the toolbar.

### `WIRE: STORAGE` — your cloud storage integration

In the script, `scheduleSync()` is where state pushes to cloud storage. Replace the stub with calls to your storage of choice (Google Drive, OneDrive, Dropbox, S3, a private API — anything your refresh task can also reach).

If you're running this through an agent framework like Claude with MCP, OpenAI with function-calling, or a custom backend with REST handlers, the exact call differs but the shape is the same: upsert `[CTRL-TWR-STATE].md` and append to `[CTRL-TWR-EDITS].log`.

Two files, both at the storage root for easy discovery by the refresh task:

- **`[CTRL-TWR-STATE].md`** — markdown with a JSON code block. Contains `closures`, `notes`, `attachments`. Live source of truth.
- **`[CTRL-TWR-EDITS].log`** — append-only audit trail, one line per UI mutation: `2026-05-12T13:04:11Z · close · platform-migration/action · "Convert the signed charter..."`. Useful for narrating "what changed since last refresh."

Debounce 2 seconds so a flurry of clicks collapses into one write.

The attachment chip's title is fetched from your storage API's metadata endpoint — Drive's `files.get`, OneDrive's `driveItem`, S3's `HeadObject`, or whatever your stack uses. That's the second WIRE point inside `addAttachment`.

### `WIRE: ASK` — your AI of choice

The Ask drawer snapshots the live dashboard (`snapshotForAsk()`) and sends it as grounding context with the question. Replace the stub with a call to whichever AI you've got — Claude (API or via MCP), OpenAI, Gemini, a local model behind an OpenAI-compatible endpoint, an enterprise search backend like Glean, or your internal LLM gateway. Anything that accepts a chat-style request works.

A small/fast model (Claude Haiku, GPT-4o mini, Gemini Flash, local 8B-class models) is usually enough — the grounding context does most of the work. Keep the system prompt strict ("answer only using the snapshot, cite the program and section by name").

## Optional: dual-mode for Chrome viewing

The real dashboard I run handles two modes: open inside an AI client with live tool access (sync to storage live) and open in plain Chrome (no tool access — sync to localStorage only, queue edits, push to storage next time the AI client is open). If you only have one mode, ignore this. If you want it, the pattern is:

1. On page load, poll for `window.<your-bridge-global>` for ~3 seconds.
2. If absent, set `body.no-bridge`. Show an info banner. Closures and notes still work — they just live in `localStorage` until handoff.
3. On next load with the bridge present, force-sync any pending localStorage edits to storage.

## State schema

```jsonc
{
  "closures":   { "<itemKey>": "2026-05-12T13:04:11Z" },     // ISO timestamp of close
  "notes":      { "<itemKey>": { "text": "…", "updatedAt": "…" } },
  "attachments":{ "<itemKey>": [ { "url": "…", "title": "…", "type": "doc|sheet|slide|file", "addedAt": "…" } ] },
  "edits":      [ { "ts": "…", "action": "close|reopen|note-set|attach|…", "prog": "…", "section": "…" } ]
}
```

`itemKey` = `<program>|<section>|<first 100 chars of line text>`. Stable enough to survive minor wording drift, specific enough to not collide across programs.

## Why this works

Three things had to combine for this to feel useful day-to-day:

1. **One page, hourly.** Not a Slack digest, not an email summary, not a project tracker — a single dashboard you scan in five minutes, refreshed often enough that you trust it.
2. **Your closures are durable.** Closing an item makes it stay closed across regenerations. That's the difference between an AI synthesis and a status report — the latter doesn't remember what you've already handled.
3. **Your notes are first-class context.** When the AI synthesizes tomorrow, your inline corrections override the source data. The dashboard learns the right answer from you, not from the documents.

## What's not in the template

- The refresh task itself — it's specific to your tool stack and your programs. The wiring guide above outlines the steps; the implementation is yours to build.
- Auth / OAuth flows for your storage and AI integrations.
- Multi-user support — this is intentionally single-user.

## License

MIT — do whatever you want with it. If you build something with it, I'd love to hear how it went.
