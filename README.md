# Win/Loss + Inventory Tracker

A collaborative, browser-based tracker for multiplayer free-for-all games. Build a collection of "documents" from an API, run games where each player brings their own documents, eliminate players in order, and track placements, per-player stats, and per-document performance. Optionally share one dataset across devices and people via Supabase.

Runs as a single static `index.html` — no build step, no server of your own to maintain. Works fully offline; cloud sync is optional.

---

## Features

- **Document collection** — search an API, bulk-import a pasted list (batched), or add items manually.
- **Multiplayer games** — add players from a roster, assign each one's documents, eliminate in order; last one standing wins. Undo, cancel, and crash-safe resume.
- **Placement tracking** — full finishing order recorded per game, editable after the fact.
- **Player roster with stable IDs** — rename freely; same-name players don't merge; history-bearing players are protected from deletion.
- **Stats** — per-type summaries, a player leaderboard (filterable by game type and date range), and document performance with baseline-adjusted **lift** and confidence indicators.
- **Charts** — cumulative wins over time, games by type, win rate by player.
- **Cloud sync (optional)** — shared workspace via Supabase, automatic on every change.
- **Data portability** — JSON backup/restore and CSV export.
- **Light/dark theme.**

---

## Quick start (local)

Download `index.html` and open it in any modern browser. That's it — everything works offline, with data saved in that browser. To share data across devices or people, set up cloud sync below.

---

## Deploying to GitHub Pages

1. Create a new **public** repository on GitHub.
2. **Add file → Create new file**, name it exactly `index.html`, paste in the full code, and **Commit changes**.
3. Go to **Settings → Pages**. Set **Source** to *Deploy from a branch*, **Branch** to `main`, folder `/ (root)`, then **Save**.
4. Wait 1–2 minutes. Your live URL appears in the Pages section: `https://yourusername.github.io/reponame/`.

Updating later: edit `index.html` in the repo and commit. Pages redeploys automatically in about a minute. Your data lives in each browser (and in Supabase if you've connected sync), so updating the code never touches it.

---

## Setting up cloud sync (Supabase)

Cloud sync lets several devices and people share one dataset. It's optional — skip this and the app still works locally.

### One-time backend setup (~3 minutes)

1. Sign up at [supabase.com](https://supabase.com) (free) and create a **New Project**. Set a database password (save it; you won't need it in the app), pick a nearby region, and wait ~2 minutes for provisioning.
2. Open the **SQL Editor**, start a **New query**, paste the following, and **Run**:

   ```sql
   create table workspaces (code text primary key, data jsonb, updated_at timestamptz);
   alter table workspaces enable row level security;
   create policy "anon all" on workspaces for all using (true) with check (true);
   ```

3. Go to **Project Settings → API** and copy two values:
   - **Project URL** — e.g. `https://abcdxyz.supabase.co`
   - **anon public** key — the long `eyJ...` string (the *anon* key, **not** service_role)

### Connecting in the app

On the live site, open **Settings → Cloud Sync** and enter:

- **Supabase URL** — your Project URL
- **Supabase anon key** — the anon public key
- **Workspace code** — a shared secret your group agrees on (e.g. `friday-night-crew-7`)

Click **Connect & Sync**. The status dot at the top turns green when connected. The first person to use a new workspace code seeds it from their local data; everyone after pulls the shared set.

### How sharing works

Anyone with the same **URL + anon key + workspace code** reads and writes the same data. Share all three with your group. Changes push automatically (debounced ~0.8s) and the app polls for others' changes every ~5 seconds.

---

## Testing after deploy

Worth checking on the live site (a sandboxed preview can't fully exercise these):

- Record a game, then **reload** — it should persist.
- Open the same URL on a **second device/browser**, enter the same three sync values, and confirm games appear. Add a game on one device; it should show on the other within ~5 seconds.
- Try **CSV export** and **JSON backup** — both trigger real file downloads.

---

## Pointing it at your own API

The demo uses the free [Scryfall](https://scryfall.com/docs/api) API (Magic: The Gathering cards) as a stand-in document source. To use your own service, edit only the clearly marked **API section** near the top of the `<script>`:

- `fetchFromAPI(query)` — search; return a list of items.
- `fetchManyByName(names)` — batch lookup by name; return `{ found, notFound }`.
- `mapItem(record)` — map one API record to the app's shape: `{ id, name, img, meta }`.

Everything downstream uses that normalized shape, so nothing else needs to change.

---

## Data model

- **players**: `{ id, name }` — stable IDs; names are display-only labels.
- **games**: `{ id, type, ts, standings: [playerId], playerDocs: { playerId: [docName] } }` — `standings` is 1st place first.
- **collection**: `{ id, name, img, meta }`.
- **typeLabels**: labels for the four game types (A–D).

State is stored in `localStorage` and, when connected, mirrored to a single Supabase row keyed by your workspace code. Backups export this whole structure as JSON (version 4).

---

## Notes & limits

- **Conflict handling is last-write-wins.** Fine for a group logging games together; not built for heavy simultaneous editing.
- **The workspace code is your access control.** Because the anon key is client-side, anyone with the key and code can read/write that table. Use a non-obvious code; don't store anything sensitive.
- **Lift is correlational, not causal**, and noisy at small samples — low-confidence rows are dimmed for this reason. Treat them as hints, not verdicts.
- **`localStorage` is per-browser** and capped (~5MB). For normal use this is plenty; very large histories with cached images could eventually hit it.
- **Supabase free tier pauses** a project after ~a week of inactivity. If sync fails after a long gap, open your Supabase dashboard once to wake it.

---

## Troubleshooting sync

A red status dot is almost always one of:

- A typo in the **URL** or **anon key**.
- The **SQL setup didn't run** — re-run the three statements above.
- The **service_role key** was used instead of the **anon** key.

The message under the Connect button hints at which. Still stuck? Check the browser console for the HTTP status code.
