---
name: rti-crawler-builder
description: Build or port indicator crawlers into the rti-platform monorepo following the canonical medallion (bronze→silver→gold) + Prefect deployment pattern. Always runs a structured intake first, asks where data lands (S3 vs Supabase), and produces a PR-ready package — crawler module + Prefect flow + prefect.yaml entry + README update + smoke run. Use whenever the user wants to (a) add a new indicator source to rti-platform, or (b) port an existing crawler from one of the legacy RTI projects (rti-australia, RTI-Australia-old, RTI-Europe, rti-energy) into the monorepo.
---

# RTI Crawler Builder

A structured, project-aware builder for indicator crawlers in the **rti-platform** monorepo. The skill knows the canonical medallion pattern (bronze→silver→gold), the Prefect deployment model (per-domain `prefect.yaml` + work pools), and the two destination layouts (Supabase Postgres for tabular bronze, AWS S3 for blob bronze with dated snapshots + `latest.json` pointer).

The goal is always **a PR-ready package** — not a snippet, not a half-implementation. Output contract: crawler module under `rti/<country>/<domain>/crawlers/`, Prefect flow under `flows/<country>/<domain>/`, deployment entry in `deploy/<domain>/prefect.yaml`, README update, plus a local smoke run that proves the wire is connected end-to-end before handing back.

---

## When this skill fires

- "Add a new indicator for the DCRI" / "Crawl Eurostat dataset X" / "Pull data from EIA endpoint Y"
- "Port the `<source>` crawler from RTI-Europe / rti-australia / rti-energy into rti-platform"
- "Build a flow that fetches `<source>` daily/monthly and writes to bronze"

Do **not** fire for: permits crawlers (use the existing `rti.*.permits.crawlers.*` patterns directly — this skill targets indicators), schema migrations, silver/gold notebook authoring (delegate to `notebook-developer` agent), or pure refactors of existing crawlers.

---

## Step 0 — Always run the intake first

Before writing a single line of code, load and follow `intake.md`. Two minutes of intake saves an hour of wrong-path work. Do not skip even if the user gave you a URL upfront — the intake captures destination, ranking, schedule, and auth that the URL alone does not.

---

## Pattern routing (after intake)

The intake outputs a `mode` (bootstrap | port) and a `style` (A | B | C). Load the matching pattern doc + example + template:

| Style | When | Load |
|---|---|---|
| **A — Generator** | Single API, you yield records as they're fetched. ~80% of indicators land here. | `patterns/crawler-anatomy.md` § A + `examples/us-indicators-egrid.md` + `templates/crawler_module.py` |
| **B — Discover / enrich** | Two-stage: a list/search call gives IDs, then per-ID detail calls. Watermarked. | `patterns/crawler-anatomy.md` § B + `examples/us-permits-accela.md` (no template — adapt from the live Accela code at `rti/us/permits/platforms/accela/`) |
| **C — Extractor + transformer** | Multi-step pipeline that lands raw JSON envelope, then cleans into a transformed JSON. RTI-Europe style. | `patterns/crawler-anatomy.md` § C + `examples/eu-indicators-eurostat.md` + `templates/style_c_extractor.py` + `templates/style_c_transformer.py` |

For Style C **first-time-in-a-domain** (e.g. first EU indicator), also bootstrap the supporting modules: `rti/eu/indicators/crawlers/base.py` (Stage/Extractor/Transformer + S3 `write_json`/`read_json` helpers) and `rti/eu/indicators/crawlers/envelope.py` (port verbatim from RTI-Europe `src/core/envelope.py`). Check if they exist before assuming a fresh write.

When in doubt, default to Style A — it's the cleanest and easiest to evolve.

---

## Destination routing (always ask, never assume)

Before writing the bronze layer, always confirm with the user where the data lands. Load `patterns/destinations.md` to walk through the trade-offs. Three patterns:

1. **Supabase Postgres** — tabular bronze, raw psycopg writes, flat columns inferred from payload. Pattern used by `rti/us/indicators/`. Best for indicators with stable row-per-state shape.
2. **AWS S3** — blob bronze, dated snapshots + `latest.json` pointer. Pattern used by `rti/eu/indicators/` (bucket: `rti-europe-195335758978`). Best for raw payloads that don't normalize cleanly to rows (JSON-stat envelopes, Excel exports, gridded climate data).
3. **Hybrid** — raw JSON to S3, normalized rows to Supabase silver via marimo notebooks. Default for EU work.

---

## Output contract

Every invocation produces:

1. **Crawler module** at `rti/<country>/<domain>/crawlers/<source>.py` (Style A) or `rti/<country>/<domain>/crawlers/<source>/{extractor,transformer}.py` (Style C). Module-level constants: `SOURCE`, `COUNTRY`, `DOMAIN`. For Style A indicators: `crawl(...)` function. For Style C: `Extractor` and `Transformer` classes.
2. **Prefect flow** at `flows/<country>/<domain>/<source>.py` with `@flow(name=...)`, `observability.configure(...)`, parameter model, and stats-dict return.
3. **Deployment entry** in `deploy/<domain>/prefect.yaml` — name, entrypoint, work_pool, schedule (only if the user gave a cadence; otherwise omit and document why).
4. **README update** — append the new source to the relevant `rti/<country>/<domain>/README.md` indicator catalog.
5. **Smoke run** — execute the crawler locally with `max_pages=1` (or equivalent dev-scoped limits), confirm at least one bronze object is written, and report the result. Do not skip this step.

Load `patterns/prefect.md` for the canonical flow + deployment YAML shapes. Load `templates/` for copy-paste-ready scaffolds.

---

## Knowledge inventory

The skill knows these RTI-family projects and their patterns:

- **rti-platform** (`~/Desktop/Labrynth/rti-platform/`) — the target monorepo. AU permits + US permits + US indicators all live here.
- **rti-australia** (`~/Desktop/Labrynth/RTI/rti-australia/`) — earlier AU permits work, predecessor to `rti/au/permits/`. Many crawlers in `rti/crawlers/{permits,stats}/`.
- **RTI-Australia-old** (`~/Desktop/Labrynth/RTI/RTI-Australia-old/`) — legacy spreadsheets + city-data findings. Not code-bearing; ignore unless explicitly asked.
- **RTI-Europe** (`~/Desktop/Labrynth/RTI/RTI-Europe/`) — DCRI-EU work, ~64 sources in `src/sources/<name>/{extractor,transformer}.py`. The canonical Style C reference.
- **rti-energy** (`~/Desktop/Labrynth/rti-energy/`) — earlier US energy/indicators work, predecessor to `rti/us/indicators/`. Some crawlers in `rti-us/crawlers/`.

Port mode reads from these projects' crawler code, normalizes to the target style, and lands in `rti-platform`.

---

## Skill anti-patterns

- **Never silently choose a destination.** S3 vs Supabase is a user decision every time. Even if the last 5 crawlers went to S3, ask.
- **Never skip the smoke run.** A crawler that compiles but doesn't actually fetch is worse than no crawler — it lulls into false confidence. Run it with limits.
- **Never commit `.env` values, API keys, or access tokens** in any generated code or doc.
- **Never invent table names or schema layouts.** Read the existing schema (`rti.us.indicators.db.bronze` for Postgres bronze, `s3://rti-europe-195335758978/eu/indicators/<source>/raw/` for S3) and conform to it.
- **Never collapse Style C into Style A** without telling the user. Style C preserves the transformer's cleaning logic as durable code; folding it into a notebook loses that. Ask before merging.
