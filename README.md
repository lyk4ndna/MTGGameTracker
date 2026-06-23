# Win/Loss + Inventory Tracker

A collaborative, browser-based tracker for multiplayer free-for-all games. Build typed **collections** of "documents" from an API, assign a collection to each player, eliminate players in order, and track placements, per-player stats, and per-document performance. Optionally share one dataset across devices and people via Supabase.

Runs as a single static `index.html` — no build step, no server of your own to maintain. Works fully offline; cloud sync is optional.

---

## Features

- **Multiple typed collections** — create named collections, each with a type (A–D). A document can live in several collections; a Type-A collection holds only one of each document, while other types allow multiples (e.g. `4 Sol Ring`).
- **Document sourcing** — search an API, bulk-import a pasted list (with quantity prefixes and exact-then-fuzzy matching), or add items manually. New additions go into a chosen target collection.
- **Multiplayer games** — pick a game type, add players, assign each a collection **of that type**; eliminate in order; last one standing wins. Undo, cancel, and crash-safe resume.
- **Players & roster** — permanent player IDs (rename freely, same-name players don't merge), with an optional **default collection** per player that auto-fills at game setup.
- **Placement tracking** — full finishing order recorded per game, editable after the fact.
- **Stats** — per-type summaries, a player leaderboard (filterable by game type and date range), and document performance with baseline-adjusted **lift** and confidence indicators.
- **Charts** — cumulative wins over time, games by type, win rate by player.
- **Cloud sync (optional)** — shared workspace via Supabase, automatic on every change.
- **Data portability** — JSON backup/restore and CSV export.
- **Light/dark theme.**

---

## Quick start (local)

Download `index.html` and open it in any modern browser. Everything works offline, with data saved in that browser. To share across devices or people, set up cloud sync below.

---

## Core concepts

### Collections
A **collection** is a named list of documents with a **type** (A–D). Create and manage them on the **Collections** tab. The type matters at game time (see below). The Type-A rule — one of each document — is enforced when adding; other types allow duplicates.

### Documents
Added from the **Browse** tab into whichever collection you've set as the target (selector at the top of Browse). Sources: API search, bulk paste, or manual entry. Unmatched bulk names are added as manual entries so nothing is lost.

### Players
Managed in **Settings → Roster**. Each has a permanent ID and an optional **default collection**. Renaming a player changes only their display name everywhere; players with game history can't be deleted (rename instead).

### Games
On the **Session** tab: choose a **game type**, add players, and assign each one a collection **whose type matches the game type**. A player's default collection is pre-selected when its type matches. Eliminate players in order; the last remaining is 1st place. Each game stores the finishing order, the collection each player used, and a snapshot of their documents.

---

## Bulk import syntax

On the Browse tab, paste one document per line into **Bulk add**:

```
4 Sol Ring
Lightning Bolt
2x Counterspell
```

- A leading number adds multiples: `4 Name`, `4x Name`, and `4 x Name` all work (capped at 99 per line).
- No number means one copy.
- Each unique name is looked up once (exact match first, then a fuzzy retry that catches typos, punctuation, and double-faced cards), then expanded by quantity.
- Names that can't be matched are added as manual entries and listed so you can fix them.
- If the target collection is **Type A**, duplicates are skipped (one of each).

---

## Deploying to GitHub Pages

1. Create a new **public** repository on GitHub.
2. **Add file → Create new file**, name it exactly `index.html`, paste in the full code, and **Commit changes**.
3. Go to **Settings → Pages**. Set **Source** to *Deploy from a branch*, **Branch** to `main`, folder `/ (root)`, then **Save**.
4. Wait 1–2 minutes. Your live URL appears in the Pages section: `https://yourusername.github.io/reponame/`.

### Updating later
Edit `index.html` in the repo and commit — Pages redeploys automatically in about a minute. Your data lives in each browser (and in Supabase if connected), so updating the code never touches it.

**Before a data-model update, export a JSON backup** (Settings → Backup & Restore) as a safety net. On first load after such an update, existing data is migrated automatically.

---

## Setting up cloud sync (Supabase)

Cloud sync lets several devices and people share one dataset. Optional — skip it and the app still works locally.

### One-time backend setup (~3 minutes)

1. Sign up at [supabase.com](https://supabase.com) (free) and create a **New Project**. Set a database password (save it; not needed in the app), pick a nearby region, and wait ~2 minutes.
2. Open the **SQL Editor**, start a **New query**, paste this, and **Run**:

   ```sql
   create table workspaces (code text primary key, data jsonb, updated_at timestamptz);
   alter table workspaces enable row level security;
   create policy "anon all" on workspaces for all using (true) with check (true);
   ```

3. Go to **Project Settings → API** and copy:
   - **Project URL** — e.g. `https://abcdxyz.supabase.co`
   - **anon public** key — the long `eyJ...` string (the *anon* key, **not** service_role)

### Connecting in the app

In **Settings → Cloud Sync**, enter the **Supabase URL**, the **anon key**, and a **workspace code** (a shared secret your group agrees on, e.g. `friday-night-crew-7`). Click **Connect & Sync**. The status dot turns green when connected. The first person to use a new code seeds it from their local data; everyone after pulls the shared set.

### How sharing works
Anyone with the same **URL + anon key + workspace code** reads and writes the same data. Share all three. Changes push automatically (~0.8s debounce) and the app polls for others' changes every ~5 seconds.

### Coordinating updates
When the app's data model changes (like the collections update), have everyone on a shared workspace update to the same version around the same time, then reload. An old version still running could overwrite the shared row with outdated-format data on its next save.

---

## Testing after deploy

Worth checking on the live site (a sandboxed preview can't fully exercise these):

- Create a collection, add documents to it, then **reload** — it should persist.
- Run a game assigning collections to players; confirm it appears in history with the right collection per player.
- Open the same URL on a **second device/browser**, enter the same three sync values, and confirm data appears and updates within ~5 seconds.
- Try **CSV export** and **JSON backup** — both trigger real file downloads.

---

## Pointing it at your own API

The demo uses the free [Scryfall](https://scryfall.com/docs/api) API (Magic: The Gathering cards) as a stand-in document source. To use your own service, edit only the marked **API section** near the top of the `<script>`:

- `fetchFromAPI(query)` — search; return a list of items.
- `fetchManyByName(names)` — batch lookup by name; return `{ items, notFound }` where `items` is aligned to the input order (`null` for a miss).
- `mapItem(record)` — map one API record to the app's shape: `{ id, name, img, meta }`.

Everything downstream uses that normalized shape, so nothing else changes.

---

## Data model

- **collections**: `{ id, name, type, docs: [{ id, baseId, name, img, meta }] }` — `baseId` is the source document ID; `id` is unique per copy so duplicates coexist. Type-A collections hold one of each `baseId`.
- **players**: `{ id, name, defaultCollId }` — stable IDs; names are display-only.
- **games**: `{ id, type, ts, standings: [playerId], playerColl: { playerId: collectionId }, playerDocs: { playerId: [docName] } }` — `standings` is 1st place first.
- **typeLabels**: labels for the four game types (A–D).

State is stored in `localStorage` and, when connected, mirrored to a single Supabase row keyed by your workspace code. Backups export this whole structure as JSON (version 5). Older backups (v1–4) are migrated on import.

---

## Notes & limits

- **Type-A + matching-type rule**: a game of a given type can only use collections of that type, and Type-A collections never hold duplicates. This is by design; it means Type-A games can't feature players running multiples of a document.
- **Conflict handling is last-write-wins.** Fine for a group logging games together; not built for heavy simultaneous editing.
- **The workspace code is your access control.** The anon key is client-side, so anyone with the key and code can read/write that table. Use a non-obvious code; don't store anything sensitive.
- **Lift is correlational, not causal**, and noisy at small samples — low-confidence rows are dimmed. Treat them as hints, not verdicts.
- **`localStorage` is per-browser** and capped (~5MB). Plenty for normal use; very large histories with cached images could eventually hit it.
- **Supabase free tier pauses** a project after ~a week of inactivity. If sync fails after a long gap, open your Supabase dashboard once to wake it.

---

## Troubleshooting sync

A red status dot is almost always one of:

- A typo in the **URL** or **anon key**.
- The **SQL setup didn't run** — re-run the three statements above.
- The **service_role key** was used instead of the **anon** key.

The message under the Connect button hints at which. Still stuck? Check the browser console for the HTTP status code.
