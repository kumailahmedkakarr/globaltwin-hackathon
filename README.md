# GlobalTwin AI — Hackathon MVP

Real-time earthquake hazard intelligence: live USGS ingestion → explainable
risk classification → AI situation summary → live map dashboard.

This is the **Hackathon MVP tier** of a larger platform vision (see
`docs/` if you brought the vision/architecture doc along). It intentionally
does one hazard type well instead of many hazard types shallowly — see
"What's deliberately not here" below.

Dark "watch room" dashboard: live map on the left, AI/template situation
summary and risk-ranked hazard feed on the right. Run it locally to see it —
the map tiles need a live internet connection to render.

---

## Run it (60 seconds, no Docker needed)

```bash
python -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate
pip install -r requirements.txt
uvicorn app.main:app --reload
```

Open **http://localhost:8000**. That's it — one process serves both the
API and the dashboard. On first startup it fetches the real USGS feed
automatically; give it a few seconds.

No API key is required for anything to work. `ANTHROPIC_API_KEY` is
optional — see below.

### Run it with Docker instead

```bash
docker compose up --build
```

Same thing, containerized. Config is via environment variables — copy
`.env.example` to `.env` first if you want to change the feed or add an
API key.

### No internet at the venue?

Load the bundled offline fixture instead of hitting the live feed:

```bash
python -c "
from app.database import init_db
from app.ingestion import ingest_from_file
init_db()
print(ingest_from_file('sample_data/sample_usgs_feed.json'))
"
uvicorn app.main:app --reload
```

The fixture is shaped exactly like the real USGS response (same fields,
same schema) — it's what the automated tests run against — but the
timestamps are static, so events will show as "over a year ago" in the
UI. That's expected for offline demo data, not a bug.

---

## Turning on the AI summary

Without an API key, the "Situation summary" panel uses a deterministic,
templated summary — the app is fully functional without any AI dependency.

To get an actual Claude-generated summary during your demo:

```bash
export ANTHROPIC_API_KEY=sk-ant-...
uvicorn app.main:app --reload
```

The summary badge switches from `TEMPLATE` to `AI` when it's live. The
model only ever narrates data already fetched from USGS and already
scored by the rule-based risk engine — it's told explicitly not to invent
locations, magnitudes, or casualty numbers (see `app/ai_summary.py`).

---

## What this actually does

1. **Ingests** the real, public, no-key-required USGS earthquake GeoJSON
   feed (`app/ingestion.py`) — default is the last day of magnitude 2.5+
   events, configurable via `USGS_FEED` in `.env`.
2. **Classifies risk** with a transparent, rule-based scorer
   (`app/risk.py`) — magnitude, USGS's own significance score, PAGER
   alert level, tsunami flag, and felt-report count each contribute a
   documented number of points, and every event carries the plain-
   language reasons for its score. This is deliberately *not* a black-box
   ML model — every point is explainable to a judge or a responder.
3. **Summarizes** the current situation in plain language, either via
   Claude or a deterministic fallback template.
4. **Displays** it all on a live map + sidebar dashboard that polls the
   backend every 60 seconds, with a manual "Refresh feed" button that
   triggers an immediate re-fetch from USGS.

### API reference

| Endpoint | Description |
|---|---|
| `GET /api/hazards` | All tracked events, sorted by risk score. Filters: `min_magnitude`, `risk_level` |
| `GET /api/hazards/{id}` | Single event |
| `POST /api/refresh` | Trigger an immediate re-fetch from USGS |
| `GET /api/summary` | Current AI/template situation summary |
| `GET /api/health` | Status, event count, active feed, refresh interval |

Interactive API docs: **http://localhost:8000/docs** (FastAPI auto-generates
this — worth showing judges directly).

---

## What's deliberately not here

Per the phased roadmap this MVP sits inside, the following are **out of
scope on purpose**, not oversights:

- **Other hazard types** (wildfire, flood, storm). The ingestion layer
  is built so adding one is a new `fetch_*` + `normalize_*` pair, not a
  rewrite — see the docstring at the top of `app/ingestion.py`.
- **PostGIS / a real spatial database.** This uses SQLite with a schema
  shaped to map directly onto a PostGIS `geography(Point, 4326)` column
  later (see `app/models.py`), so swapping storage is a config change,
  not an app rewrite.
- **Automated dispatch or public alerting.** All output here is
  advisory and informational. No AI output in this codebase triggers an
  action — it only classifies and summarizes what already happened.
- **User accounts / auth.** Single-tenant, read-only public API for the
  demo.

## Project layout

```
app/
  main.py         FastAPI app, mounts static frontend + API routes
  config.py       Env-based settings, all optional
  models.py       SQLModel schema (PostGIS-shaped)
  database.py     Engine/session setup
  ingestion.py    USGS fetch + normalize + upsert
  risk.py         Explainable rule-based risk scorer
  ai_summary.py   Claude summary with deterministic fallback
  scheduler.py    Background periodic refresh
  routers/hazards.py   API endpoints
static/
  index.html, style.css, app.js   Dashboard (vanilla JS + Leaflet, no build step)
  vendor/leaflet/  Leaflet vendored locally — no CDN dependency at demo time
tests/            pytest — risk engine + feed normalization
sample_data/      Offline fixture, same schema as the live feed
```

## Tests

```bash
pip install pytest
python -m pytest tests/ -v
```
