# CLAUDE.md — House_Prices

Guidance for AI coding assistants (Claude Code and others) working in this repo.
Read this file first, then `docs/spec.md` (the source of truth), then `docs/handoff.md` (milestones).

## What this project is

A learning data-engineering project that behaves like a real company pipeline:
a synthetic Polish housing market (OLX-like and Otodom-like listings) is served through a
**mock listings API**; an **adapter** crawls that API daily and lands raw data in a
**data lake** (bronze); Databricks turns it into clean history (silver, SCD2) and
analytics (gold). The design goal: anyone can swap the mock API for a real scraper or
provider by changing **only the adapter, one parser and config**.

## Hard rules

- **Bronze is append-only and raw.** Never clean, rename or type the payload in adapters; the only exception is stripping tracking parameters from `source_url`. Parsing happens in silver.
- **Never read `truth/` from pipeline code.** It is ground truth for tests only.
- **No secrets in code, config or chat.** Use environment variables / GitHub Secrets:
  `DATABRICKS_HOST`, `DATABRICKS_TOKEN`, `MOCK_API_KEY`.
- **No real scraped data in the repo.** Only synthetic fixtures.
- **Determinism:** same `seed` + same date ⇒ byte-identical generator output. Never use unseeded randomness or wall-clock time inside the generator.
- **Personal data:** seller names, phones, emails may exist in raw payloads; they are dropped or hashed in silver and never appear in gold.
- **Scope filter only in gold.** Bronze and silver hold all of Poland; Warsaw is selected via `analysis_scope` in `config/pipeline.yaml`.

## Team and platforms

- 3+ contributors: one on macOS, others on Windows. Everything must work natively on both
  (no WSL, no Docker required).
- Use `pathlib` for paths, never hard-coded `/` or `\\`. Text files are UTF-8 with LF (`.gitattributes`).
- Polish text keeps diacritics (Łódź, Białołęka). Generated descriptions and labels are in **English**.

## Tooling

- Python 3.12, managed with **uv** (`uv sync`, `uv run ...`).
- Lint/format: **ruff**. Tests: **pytest**. Hooks: **pre-commit** (ruff, end-of-file, secret scan).
- Mock API: **FastAPI** + uvicorn. HTTP client: **httpx**. Config: YAML + pydantic models.
- Databricks: Free Edition, Unity Catalog, Delta, Auto Loader. Notebooks are `.py` source files in `databricks/`.
- CI: GitHub Actions (`.github/workflows/ci.yml`), daily crawl (`daily.yml`).

## Common commands

```bash
uv sync                                   # install
uv run pytest                             # tests
uv run ruff check . && uv run ruff format .
uv run mock-api                           # start mock API on http://localhost:8000
uv run crawl --mode daily                 # adapter: crawl API -> data/raw/...
uv run crawl --mode backfill              # load the last 30 days via as_of
uv run upload                             # push data/raw to the Databricks Volume
```

## Conventions

- Small PRs, one milestone step each; every PR has tests. `main` is protected (1 review).
- Commit style: `feat:`, `fix:`, `test:`, `docs:`, `chore:`.
- Public functions have type hints and a one-line docstring.
- New decisions that are hard to reverse get an ADR in `docs/adr/` (never edit an accepted ADR; supersede it).
- When a change touches the bronze contract, bump `schema_version` and update `docs/spec.md` in the same PR.

## Teaching mode

Some contributors are learning. When asked to explain, explain the *why* before the *how*,
and prefer guiding with steps and hints over writing the full solution, unless the person
explicitly asks for the code.
