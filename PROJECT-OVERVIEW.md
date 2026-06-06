# Screening Room — Project Overview

A premium, multi-user **life tracker**. It started as a movie & TV tracker and is being
extended into a "worlds" app — Entertainment first, then Dining, Places, Theater, and more.
Users sign in, browse curated/auto-updated catalogs, track what they've experienced, and get
an **implicit rating** from a few quick questions instead of fiddling with stars.

- **Live app:** https://screening-room1.netlify.app
- **GitHub repo:** https://github.com/r7verma/screening-room1
- **Look & feel:** dark cinematic glassmorphism, animated "Hub" home screen → tap a world → browse & track.

---

## 1. How it works (architecture in one breath)

```
                         ┌─────────────────────────────┐
   GitHub repo  ──push──▶│ Netlify (static hosting)    │──serves──▶  Browser (the app)
   (index.html,          │ continuous deploy on commit │                 │
    catalog.json,        └─────────────────────────────┘                 │
    dining-catalog.json,            ▲                                     │
    scripts/, .github/)             │ daily commit                       │ at runtime:
        │                           │                                     │  • reads catalog.json
        │  GitHub Action (cron)─────┘                                     │  • Supabase Auth (login)
        │  fetches TMDB, rebuilds catalog.json                            │  • Supabase DB (your library)
        ▼                                                                 │  • TMDB API (live search)
     TMDB API                                                  Supabase ◀─┘  • Google Places API (dining)
```

- **Front end:** one self-contained `index.html` (vanilla HTML/CSS/JS, no build step).
- **Hosting:** Netlify, auto-deploys on every push to `main`.
- **Catalog data:** `catalog.json` for movies/TV (refreshed daily by GitHub Action); `dining-catalog.json` for the static fallback restaurant list.
- **Auth + user data:** Supabase (hosted Postgres + Auth). RLS enforces per-user privacy.
- **Live search (entertainment):** TMDB API called directly from the browser.
- **Live search + discovery (dining):** Google Maps JS API with `PlacesService` (no CORS issues).

---

## 2. Tech stack

| Layer            | Choice                                                                  |
|------------------|-------------------------------------------------------------------------|
| Front end        | Vanilla HTML/CSS/JS, single file (`index.html`), no framework           |
| Fonts/anim       | Google Fonts (Oswald + Inter), pure CSS animations                      |
| Auth + DB        | Supabase (Postgres + GoTrue auth), `@supabase/supabase-js@2` via CDN   |
| Movie/TV data    | TMDB API                                                                |
| Dining discovery | Google Maps JS API (`libraries=places`) — `PlacesService` for textSearch / nearbySearch |
| Hosting          | Netlify (Git continuous deployment)                                     |
| Automation       | GitHub Actions (daily cron) + Node 20 script                            |

No build tooling — edit the files, commit, Netlify redeploys.

---

## 3. Repository structure

```
screening-room1/
├─ index.html                 # the entire app (UI + logic + config)
├─ catalog.json               # movies & TV catalog (auto-grown daily by GitHub Action)
├─ dining-catalog.json        # 20 hardcoded NYC restaurants (static fallback)
├─ PROJECT-OVERVIEW.md        # this file — shared context for collaborators + Claude
├─ scripts/
│  └─ update-catalog.mjs      # daily TMDB fetcher (run by the Action)
└─ .github/
   └─ workflows/
      └─ update-catalog.yml   # daily cron (06:00 UTC) + manual trigger
```

---

## 4. API keys (already in index.html)

```js
const SUPABASE_URL = "https://jxqtggajabevmpzpxhtk.supabase.co";
const SUPABASE_KEY = "sb_publishable_TJ8ChS2A0qlBNPE6JwsKnA_b382P3AR";
const TMDB_KEY     = "f55148eb9c5f84d935845bc4510abe6e";
const GOOGLE_PLACES_KEY = "AIzaSyAD3vr_u-0NJkRURtf8nxWbXeil-WtJviI";
```

Google Maps JS API is loaded at the bottom of `<body>`:
```html
<script async defer src="https://maps.googleapis.com/maps/api/js?key=AIzaSy...&libraries=places&callback=initGoogleMaps"></script>
```

**To run locally:** add `localhost:8080/*` as an allowed HTTP referrer on the Google Cloud API key, or Places calls will be silently blocked. Use `python3 -m http.server 8080` to serve the file.

---

## 5. Features

### Hub (home screen)
- ~2-second cinematic title reveal, then category "world" icons spring in (staggered).
- Custom gradient icon medallions per world, with hover glow.
- **Entertainment** and **Dining** are live; Places / Theater / Music / Books open "coming soon" teaser pages.

### Authentication
- Email + password and **magic link** (passwordless) — working.
- Google and Apple sign-in buttons built; need provider configuration in Supabase to activate.

### Entertainment world
- **Discover:** TMDB poster grid from `catalog.json`, sorted by popularity.
- **Live search:** queries TMDB directly.
- Filter by type (All / Movies / TV) and genre.
- Track as **Watchlist / Watching / Watched**.
- **Implicit rating:** "Watched" opens 4-question modal → 0–10 score (no stars).
- **My Library:** grouped by genre, sorted by rating.
- **Stats:** counts, avg rating, taste profile bar chart.

### Dining world
- **Discover tab** has three modes driven by Google Places API:
  - **🔥 Trending** — `nearbySearch` with `RankBy.PROMINENCE` around Times Square (40.758, -73.9855), radius 3 km, sorted by Google rating. Falls back to `dining-catalog.json` if API fails.
  - **👥 Friends** — currently shows catalog sorted by user's own ratings (friends feature not yet built).
  - **All** — catalog fallback; if location is available, sorted by distance.
- **Search bar** — typing + Enter fires `placesService.textSearch` appending `"restaurant NYC"` if needed. Returns up to 20 live Google Places results.
- **📍 Near Me button** — geolocation → `nearbySearch` within 1 mile (1609 m), sorted by distance.
- **Beli-style row cards** — cuisine-colored gradient swatch, restaurant name, cuisine · neighborhood · price, Google ★ rating + review count (formatted as `2.1k`), distance if location known, tag chips, score ring (filled when rated), status dot (Visited / Going / Want).
- Cards highlight with a gold border on hover.
- Track as **❤️ Want / 📅 Going / ✅ Visited**. "Visited" opens the rating modal.
- **Implicit rating:** 4 questions — food (0–10), vibe (0–3), value (0–2), would return (0–2) → score out of 17. Displayed on the score ring.
- **My List tab:** sorted by rating (Visited), shows items from Supabase. Looks up card data from `dCurrentResults` cache (Google Places results from the session) or `dining-catalog.json`.
- **Stats tab:** Visited / Going / Want counts, avg score (displayed as /10 by converting `/17 × 10`), top pick, cuisine bar chart.

### Shared engine
- Both verticals use Supabase `tracked_items` table.
- `dCurrentResults` is a session cache that holds the last-loaded Google Places results — needed so My List can render cards for Google Places restaurants that aren't in the static catalog.

---

## 6. Database (Supabase)

Run `supabase-setup.sql` once. Key table:

**`tracked_items`**

| Column       | Notes                                                                 |
|--------------|-----------------------------------------------------------------------|
| `user_id`    | FK to `auth.users`; RLS: `user_id = auth.uid()`                      |
| `vertical`   | `"entertainment"` or `"dining"`                                       |
| `tmdb_id`    | For entertainment: TMDB integer ID. For dining: Google `place_id` string (e.g. `ChIJabc…`). **Must be `text` type — see note below.** |
| `media_type` | `"movie"`, `"tv"`, or `"restaurant"`                                  |
| `title`      | Display name                                                          |
| `genres`     | `text[]` — for dining, first element is the cuisine                   |
| `status`     | Entertainment: `want`/`watching`/`watched`. Dining: `want`/`visiting`/`visited`. |
| `rating`     | Numeric. Entertainment: 0–10. Dining: 0–17.                           |
| `answers`    | `jsonb` — raw question answers                                        |
| `rated_at`   | Timestamp of rating                                                   |
| `updated_at` | Timestamp of last status change                                       |

Unique constraint: `(user_id, tmdb_id, media_type)`.

> ⚠️ **`tmdb_id` must be `text` type** — Google Place IDs are strings like `ChIJabc123…`. If your column is `integer`/`bigint`, run this once in the Supabase SQL Editor:
> ```sql
> ALTER TABLE tracked_items ALTER COLUMN tmdb_id TYPE text USING tmdb_id::text;
> ```

---

## 7. The daily catalog automation

- `.github/workflows/update-catalog.yml` runs `scripts/update-catalog.mjs` at **06:00 UTC** daily.
- Fetches many pages of TMDB (popular, top-rated, now-playing, upcoming, trending, discover), merges into `catalog.json`, commits, Netlify redeploys.
- Requires a repo Actions secret named `TMDB_KEY`.
- GitHub pauses scheduled workflows after ~60 days of inactivity — click **Run workflow** to re-arm.

---

## 8. Setup / running locally

### Local dev
```bash
cd /path/to/screening-room1
python3 -m http.server 8080
# open http://localhost:8080
```
- Email/password login works fine locally.
- OAuth (Google/Apple) redirects back to the Netlify URL — use email login for local testing.
- Google Places API: add `localhost:8080/*` as an allowed HTTP referrer in Google Cloud Console → Credentials → your API key.

### One-time Supabase setup
1. Run `supabase-setup.sql` in SQL Editor.
2. Authentication → Settings: turn off "Confirm email" for instant email/password login.
3. Authentication → URL Configuration: set Site URL + Redirect URLs to `https://screening-room1.netlify.app`.
4. Alter `tmdb_id` to `text` if not already (see section 6).

### GitHub Action
Add `TMDB_KEY` as a repo Actions secret. Run the workflow once to populate `catalog.json`.

---

## 9. Known limitations

- **`dCurrentResults` is session-only** — Google Places results found in one session aren't cached to disk. My List works for that session; on refresh, only items in `dining-catalog.json` get full card rendering (others fall back to name/cuisine from Supabase).
- **Friends feature** not built — the 👥 tab currently shows the catalog sorted by the user's own ratings.
- **Dining catalog** (`dining-catalog.json`) is manual — no daily refresh Action yet.
- **Google/Apple login** need provider configuration in Supabase before they work.
- **API key domain restriction** — the Google Places key needs `localhost:8080/*` added for local dev (remove after testing).
- Single-file architecture is intentional for simplicity — consider splitting into modules if the file grows much beyond ~1500 lines.

---

## 10. Roadmap / next tasks

- **Persist Google Places data** — cache place details to Supabase or a `places-cache.json` so My List renders correctly across sessions.
- **Friends feature** — show what friends have rated / saved (requires a `friendships` table).
- **Neighborhood filter** — add to dining discover controls.
- **More cities** — right now Trending/Near Me defaults to NYC; add a city picker.
- **Daily dining refresh** — a GitHub Action that updates `dining-catalog.json` with curated additions.
- **Profile/settings screen** — cross-device already works (Supabase), just needs a UI.
- **Next world** — Places or Theater on the same engine.

---

## 11. Quick glossary for a new collaborator

- **Vertical / world:** a top-level category (Entertainment, Dining, …).
- **Implicit rating:** score derived from a short questionnaire — no manual star input.
- **catalog.json:** the static movie/TV browse data, refreshed daily by the GitHub Action.
- **dining-catalog.json:** 20 hardcoded NYC restaurants used as a fallback when Google Places is unavailable.
- **dCurrentResults:** in-memory array of the last batch of Google Places results — used so status buttons and My List can look up card data within a session.
- **PlacesService:** the Google Maps JS API class used for `textSearch` (keyword search) and `nearbySearch` (location-based). Loaded via script tag with `libraries=places&callback=initGoogleMaps`.
- **The Action:** the GitHub Actions cron that keeps `catalog.json` fresh from TMDB.
- **Score ring:** the SVG circle on each dining card that fills proportionally to the user's rating (0–17 scale).
