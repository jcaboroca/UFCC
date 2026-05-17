# UFCC — Unofficial Football Club Championship

Lineage tracker for the *Unofficial Football Club Championship*: a single
imaginary title that passes to whichever club beats the current holder, every
single match, since **11 Nov 1871** (Wanderers 0–0 Harrow Chequers).

Live: <https://ufcc-stats.github.io/UFCC/>

Historical data scraped once from
[stevesfootballstats.uk](https://www.stevesfootballstats.uk/unofficial_football_club_championship_ufcc.html).
Daily-ish updates pulled from [football-data.org](https://www.football-data.org/)
(free tier).

---

## Web

Static, vanilla HTML/CSS/JS, served from `/docs` via GitHub Pages. No build
step. VS Code–style activity bar with seven lazy-loaded views:

- **Feed** — reverse-chronological infinite scroll of every title match, with a
  draggable year timeline on the right. Hero on top shows the current champion,
  days held, the match where the reign started, and the **next scheduled match**
  (opponent, kickoff in the user's locale, competition, home/away).
- **Rankings** — all-time clubs sorted by matches defended (with a filter box).
- **Longest reigns** — top 100 individual reigns by matches defended.
- **Countries** — clubs grouped by country (deduced from the dominant
  competition of each club, with bbox of coordinates as fallback).
- **Map** — Leaflet map (lazy-loaded from unpkg) with one circle per club, sized
  by total days as champion.
- **Stats** — totals, current champion, longest/shortest reign, most-active year.
- **Search** — multi-term filter over every match in history.

Cross-browser tested on Chrome and Safari (incl. the `<input type="search">`
reset and the `position: fixed` sidebar). Mobile: the activity bar collapses
into a horizontal tab bar.

### Metrics — why matches, not days

The primary ranking metric is **matches defended**, not days held. Real
championship tenure is interrupted by the off-season: a club crowned in May and
beaten in the first matchday of August looks dominant on the calendar but only
defended a couple of matches. Days are still shown as a secondary number.

Reign duration also **excludes the two World Wars** (1914-08-01 → 1919-08-31 and
1939-09-01 → 1946-08-31), when European league football was suspended.

---

## Data pipeline

### One-shot historical scrapers (already run, output committed)

```sh
python3 scrape_champions.py    # ~456 historical champions from KML (name, lat, lon)
python3 scrape_lineage.py      # ~5400+ matches with the derived champion_after each one
python3 fetch_crests.py        # ~1000 club crests
```

### Daily incremental updates

```sh
FOOTBALL_DATA_KEY=... python3 update_from_api.py
python3 export_web.py
```

`update_from_api.py`:

1. Reads the current champion from the DB.
2. Asks football-data.org for that team's `FINISHED` matches after the last one
   we already have.
3. For each new match: inserts it, recomputes the reign, swaps the champion if
   the holder lost. Repeats with the new champion (up to 10 swaps per run).
4. Asks for the next `SCHEDULED/TIMED` match of the current champion and writes
   `docs/data/next_match.json`.

`export_web.py` turns the SQLite DB into the JSON files the web reads:

| File | Contents |
| --- | --- |
| `matches.json` | Every match, newest first |
| `clubs.json` | Crest filename per club |
| `years.json` | Index of each year in the matches list (for the timeline) |
| `rankings.json` | Clubs sorted by matches defended (with days, reigns, crest) |
| `longest_reigns.json` | Top 100 individual reigns |
| `countries.json` | Countries with top clubs per country |
| `champions_geo.json` | Lat/lon + days for the map |
| `stats.json` | Global stats card |
| `next_match.json` | Next scheduled match of the current champion |

Storage: `ufcc.db` (SQLite). Tables: `matches`, `clubs`, `champions`, `reigns`.

---

## Automation (GitHub Actions)

`.github/workflows/update-champion.yml` runs every 5 minutes
(`*/5 * * * *`). Most runs exit in milliseconds without touching the API thanks
to a smart skip in `update_from_api.py`. It only does real work when:

- **Minute :00 of any hour** — base hourly check, guarantees ≤1 h freshness no
  matter what.
- **Inside `[kickoff + 95 min, kickoff + 5 h]`** of the next match recorded in
  `next_match.json`. 95 minutes covers 90' + added time; the 5 h tail covers
  extra time, penalties and broadcast/API delay.
- **`FORCE_UPDATE=1`** is set, also wired as a `workflow_dispatch` input
  ("Run workflow" → `force: 1` in the Actions UI).

Result: the post-match update lands within ~5 min of full time; the rest of the
day there are zero API calls. Worst-case daily usage is **~55 API calls** on
match days, well under the free-tier ceiling of 14 400/day.

### What happens when…

- **Next match has no published kickoff time yet** → the JSON keeps whatever
  placeholder the API returns; the hourly `:00` check refreshes it as soon as
  the real time is published.
- **No matches in the calendar** → `next_match.json` becomes `null`; the hero's
  "Next match" pill is hidden; the hourly check keeps trying.
- **Champion loses in a fixture that wasn't in `next_match.json`** (e.g. a cup
  game added late) → reflected within the next `:00` (≤1 h delay).
- **Match postponed/cancelled** → drops out of `SCHEDULED/TIMED`; the next
  hourly check rewrites `next_match.json` with the following fixture.
- **Match goes to extra time + penalties** → still inside the 5 h window.

`FOOTBALL_DATA_KEY` is stored as a GitHub Actions secret on the repo.

---

## Repo layout

```
docs/                 # GitHub Pages root
├── index.html        # single-page app entry
├── app.js, app.css   # vanilla JS + CSS
├── crests/           # club crest images (PNG)
└── data/             # generated JSONs (committed)
ufcc.db               # SQLite database (committed)
scrape_*.py           # one-shot historical scrapers
update_from_api.py    # incremental updater (football-data.org)
export_web.py         # SQLite → JSON exporter
fetch_crests.py       # one-shot crest scraper
.github/workflows/
└── update-champion.yml
```

---

## Sources

- Historical lineage: <https://www.stevesfootballstats.uk/unofficial_football_club_championship_ufcc.html>
- Live updates: <https://www.football-data.org/>
- Map tiles: CartoCDN dark (via Leaflet 1.9.4)
