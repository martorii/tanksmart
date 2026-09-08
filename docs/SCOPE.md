# TankSmart — Project Scope

**Status:** Draft, pre-implementation
**Last updated:** 2026-09-08

---

## 1. Overview

TankSmart is a personal analytics project over German fuel price data. It builds a
local historical database of fuel prices for all German filling stations and exposes
it through a dashboard that answers one core question:

> **When is the best time to refuel at the stations I care about?**

It is a batch, single-user, local-first analytics project. It is not a service.

---

## 2. Goals and non-goals

### Goals

- Load the complete historical fuel price dataset for **all of Germany** into a local
  analytical database.
- Explore and visualise price behaviour per station, fuel type, hour of day and day of
  week.
- Produce shareable reports and charts (intraday price profiles, station comparisons,
  savings estimates).
- Keep the door open for price forecasting later, without designing for it now.

### Non-goals (explicitly out of scope)

These were live options during scoping and were deliberately ruled out:

| Excluded | Reason |
| --- | --- |
| Tankerkönig **live API** polling | Historical bulk dump supersedes it; no real-time requirement. No API key needed. |
| A 24/7 ingestion service | Data refresh is a manual/cron batch command, not a daemon. |
| Server infrastructure (VPS, cloud) | Runs locally. No always-on component. |
| PostgreSQL / TimescaleDB | Over-engineered without concurrent writers. See §4. |
| FastAPI backend | Streamlit talks to the database directly. |
| User accounts, auth, GDPR handling | Single user. Favourites live in a config file. |
| Commercial use | Dataset licence is non-commercial (see §3). |

Consequence of dropping live data: TankSmart can answer *"what time of day is cheapest
at this station"* but **not** *"is it cheap right now"*. That is accepted.

---

## 3. Data source

### Primary source

Tankerkönig publishes the complete history of every fuel price change for every German
filling station since 2014, as daily CSV files in a public Git repository:

```
git clone https://tankerkoenig@dev.azure.com/tankerkoenig/tankerkoenig-data/_git/tankerkoenig-data
```

- **Granularity:** event-level. One row per price change, with an exact timestamp.
  Not periodic snapshots.
- **Coverage:** all German stations (order of ~15,000), fuel types diesel / E5 / E10.
- **Size:** ~20 GB for the full history since 2014 (~2 GB per year).
- **Refresh:** new data appended nightly for the previous day. A daily `git pull` keeps
  the local copy current. Skipping it simply freezes the data; nothing breaks.

Event-level data is strictly better than the 30-minute polling originally considered:
30-minute sampling would systematically miss short intraday price troughs, which are
exactly what the project wants to detect.

### Licence — action required

- The Tankerkönig **API** data is published under **CC BY 4.0**.
- The **historical bulk dataset** is referenced as **CC BY-NC-SA 4.0**
  (attribution + non-commercial + share-alike).

Fine for personal use. **Before publishing any chart, dashboard or derived dataset
publicly, verify the current terms on the official Tankerkönig site and include the
required attribution.** This has not been verified against the official terms page yet.

### Possible future source

Forecasting price *levels* (as opposed to intraday shape) requires an exogenous signal —
Brent crude or wholesale fuel prices. Not in scope; see §7 Phase 4 and §8.

---

## 4. Architecture

```
Azure DevOps git repo (CSV)
        |  sync      git clone / git pull
        v
  data/raw/                     untouched CSV files
        |  load      parse, type, normalise wide -> long, UTC -> Europe/Berlin
        v
  data/staging/*.parquet        partitioned by year/month
        |  build     SQL transforms
        v
  data/prices.duckdb            core layer: price_changes, price_intervals, stations
        |  aggregate SQL rollups
        v
  data/marts.duckdb             aggregates only (tens of MB, shareable)
        |
        v
  Streamlit + Plotly            dashboard
```

### Why DuckDB rather than PostgreSQL/TimescaleDB

Postgres + TimescaleDB is the right answer for a 24/7 poller with concurrent writers.
With no server and no concurrent writes it only adds a component to administer. DuckDB
is a single file, reads Parquet and CSV natively, is columnar (well suited to
"median diesel price by hour of day over two years"), and has no operational overhead.

**Revisit this decision if** multiple users ever need to write concurrently through a
backend — that is the case where Postgres wins.

### Two-database split

`prices.duckdb` (full detail, gigabytes) stays local. `marts.duckdb` (aggregates only,
tens of megabytes) is small enough to publish — e.g. to Streamlit Community Cloud, which
will not host a multi-gigabyte database. Building this split from the start avoids a
painful retrofit, even though nothing is published today.

### Stack

| Layer | Choice | Rationale |
| --- | --- | --- |
| Language | Python 3.12, `uv` for dependencies | `uv` is substantially faster than pip/poetry here |
| Ingest / transform | DuckDB SQL, Polars where needed | DuckDB reads the CSVs directly; no row-by-row Python ETL |
| Storage | DuckDB + Parquet | Zero ops, columnar, good compression |
| Dashboard | Streamlit + Plotly | Interactive time series, filters, tables, export |
| Orchestration | `typer` CLI, optional cron | No Airflow/Prefect at this scale |
| Quality | `ruff`, `mypy`, `pytest` | |

---

## 5. Data model

The source files record **price changes**, not periodic observations. This has one
consequence that must be handled correctly or every downstream analysis is wrong:

> To answer "what did diesel cost at station X last Tuesday at 08:00", filtering is not
> enough. It requires an **as-of join**: the most recent change at or before that
> instant. A price stays in force for hours or days after its change event.

### Tables

- **`stations`** — dimension: uuid, brand, name, address, latitude/longitude, postcode.
  Open question: keep history of the dimension (stations open, close and rebrand) or
  current state only. See §8.
- **`price_changes`** — event-level facts: `station_uuid`, `changed_at`, `fuel_type`,
  `price`. Normalising from wide (one row carrying diesel/e5/e10) to long (one row per
  fuel type) makes everything downstream simpler.
- **`price_intervals`** — derived: `station_uuid`, `fuel_type`, `valid_from`, `valid_to`,
  `price`. Precomputing intervals once turns expensive point-in-time queries into cheap
  range lookups.
- **`fct_hourly_profile`** — mart: median and p25 price by station x fuel type x weekday
  x hour. **This table is the "best time to refuel" report.**

---

## 6. Repository layout

```
tanksmart/
|-- src/tanksmart/
|   |-- cli.py                  # tanksmart sync | load | build | dashboard
|   |-- sources/tankerkoenig.py # clone/pull, discover new files
|   |-- loading/                # CSV -> Parquet
|   |-- sql/                    # versioned SQL models
|   `-- analytics/              # profiles, forecasting
|-- dashboard/                  # Streamlit app
|-- tests/
|-- data/                       # gitignored
`-- docs/SCOPE.md
```

---

## 7. Roadmap

### Phase 0 — Exploration

Clone one month of data. Verify the real schema, timezone, change flags, outliers and
actual volume.

**Everything in §5 is a hypothesis until this phase confirms it.** Do not build the
pipeline first.

### Phase 1 — Pipeline

`sync`, `load` and `build` CLI commands. CSV -> Parquet -> DuckDB, idempotent and
resumable. Tests against a small fixture.

### Phase 2 — Dashboard v1

Station search by radius, price time series, comparison across favourite stations
(favourites in a config file, no login).

### Phase 3 — Reports

Hour-of-day x day-of-week heatmap. "How much do you save refuelling at 20:00 versus
08:00." Ranking of nearby stations by average price.

### Phase 4 — Forecasting

**Phase 3 needs no machine learning.** German intraday price behaviour is a stable
pattern (prices rise in the morning and fall through the evening); a median by
hour x weekday captures it better than a model would.

Reserve ML for forecasting the price *level* a week ahead — and note that the
Tankerkönig history alone is insufficient for that. It requires an exogenous variable
(Brent, wholesale prices), which is an additional data source and a separate decision.

---

## 8. Open questions

| # | Question | Recommendation |
| --- | --- | --- |
| 1 | How many years to load? Full history is ~20 GB. | Start with **2 years** — enough for seasonality and weekly patterns. Confirm available disk space if the full history is wanted later. |
| 2 | Which fuel types? The dump carries diesel, E5 and E10. | E5 and E10 are distinct products with distinct price dynamics. Decide: all three, or diesel + E5. |
| 3 | What does "best time to refuel" mean precisely? | Three different reports: *time of day*, *day of week*, or *wait N days*. Pick one for Phase 3. |
| 4 | Accept an external source (Brent / wholesale) for level forecasting? | Without it, a weekly forecast is little better than a recent average. |
| 5 | How are "my stations" defined? | Fixed list in a config file, or radius search from home/work in the dashboard. |
| 6 | Will the dashboard ever be shared over the internet? | Determines whether the lightweight `marts` layer is built now. Currently assumed yes, built from the start. |
| 7 | Refresh manually or via daily cron? | Either; the CLI supports both. |
| 8 | Keep station dimension history? | Stations rebrand and close. Current state only is simpler; history is needed for accurate long-range brand analysis. |

---

## 9. Known pitfalls

Verify each of these during Phase 0.

1. **Timezone.** If the dump is in UTC and is treated as local time, "the best hour to
   refuel" comes out shifted, and daylight saving introduces a one-hour error for half
   the year. Pin `Europe/Berlin` explicitly.
2. **Change flags and zero prices.** The CSVs carry change-type columns whose exact
   semantics must be confirmed against the real data. Prices of `0` and other implausible
   values must be filtered, or detected minima will be garbage.
3. **As-of joins.** See §5. The single most likely source of silently wrong results.
4. **Volume estimates.** Row counts and file sizes quoted in this document are estimates
   from public documentation, not measurements. Phase 0 replaces them with real numbers.

---

## 10. Decision log

| Decision | Choice | Date |
| --- | --- | --- |
| Geographic coverage | All of Germany | 2026-09-08 |
| Audience | Personal use, with shareable reports and charts | 2026-09-08 |
| Data acquisition | Official historical dump only; no live API | 2026-09-08 |
| Deployment | Local, no always-on ingestion process | 2026-09-08 |
| Database | DuckDB + Parquet | 2026-09-08 |
| Frontend | Streamlit + Plotly | 2026-09-08 |
