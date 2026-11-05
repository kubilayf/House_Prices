# House_Prices

A learning data-engineering project that behaves like a real company pipeline. A synthetic
Polish housing market is served through a mock listings API; an adapter crawls it daily into a
data lake (bronze), and Databricks turns that into clean history (silver) and analytics (gold).

**Status: scaffold.** The package modules are empty, so the pipeline does not run yet.

Start with [CLAUDE.md](CLAUDE.md) for the project rules and [docs/spec.md](docs/spec.md) for the
technical spec. Decisions are recorded in [docs/adr/](docs/adr/).

## Setup

Everything runs natively on macOS and Windows, with no WSL or Docker. You need
[Git](https://git-scm.com/downloads) and [uv](https://docs.astral.sh/uv/). uv installs
Python 3.12 for you.

### macOS

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh    # or: brew install uv
git clone <repo-url> House_Prices
cd House_Prices
uv sync
uv run pre-commit install
uv run pytest
```

### Windows (PowerShell)

```powershell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
# Close and reopen PowerShell so that uv is on your PATH.
git clone <repo-url> House_Prices
cd House_Prices
uv sync
uv run pre-commit install
uv run pytest
```

The first commit after `pre-commit install` takes longer, because the hooks download their tools
(including a Go toolchain for the secret scan, if you don't have one).

## Everyday commands

These are the same on both platforms:

```bash
uv run pytest                  # run the tests
uv run ruff check .            # lint
uv run ruff format .           # format
uv run pre-commit run -a       # run every hook on every file
```

`uv run mock-api`, `uv run crawl` and `uv run upload` are listed in CLAUDE.md but don't exist
yet. They will be added along with the code they run.

## Secrets

Never put secrets in code, config or chat. Set them as environment variables locally and as
GitHub Secrets in CI: `DATABRICKS_HOST`, `DATABRICKS_TOKEN`, `MOCK_API_KEY`.

```bash
export MOCK_API_KEY=...        # macOS
```

```powershell
$env:MOCK_API_KEY = "..."      # Windows
```

## Layout

```
config/                  generator and pipeline settings
docs/                    spec and decision records (ADRs)
src/house_prices/
  contract/              bronze envelope, crawl manifest, canonical listing
  generator/             seeded world, daily events, source renderers, truth export
  mock_api/              FastAPI app with OLX-like and Otodom-like endpoints
  adapters/              source adapters and templates for real sources
  runner/                shared crawl loop and bronze file writing
  parsers/               payload to canonical listing, one per source
  upload.py              push data/raw to the Databricks Volume
tests/
data/, truth/            generated locally, never committed
```
