# House_Prices — Technical Spec

Source of truth for the pipeline. Condensed from the team's design docs (Nov 2026).

## 1. Decisions so far (candidate ADRs)

| # | Decision | Why |
|---|---|---|
| 0001 | Synthetic data served by a mock API, no scraping in the public repo | Legal, reproducible, publishable; real sources plug in via adapters |
| 0002 | Bronze contract: raw envelope + crawl manifest, append-only | Any source works downstream without changes |
| 0003 | Generate all of Poland; filter to Warsaw only in gold via `analysis_scope` | Scale to other cities by config |
| 0004 | Load only the last 30 days at start, then daily | Fits Databricks Free Edition limits; extend later |
| 0005 | GitHub Actions + Databricks Jobs for scheduling, no Airflow | Free, nothing to host |
| 0006 | English text, Polish geography and PLN | Readable for all, realistic market |
| 0007 | Handbook and docs in GitHub, chat in Discord | Searchable history for late joiners |

## 2. Architecture

```
generator (seeded world) ──> mock API (FastAPI, OLX-like + Otodom-like)
        │                          │  HTTP: pagination, API key, 429, 5xx, as_of
        ▼                          ▼
   truth/ (tests only)       adapter + runner ──> data/raw/... (bronze files + manifests)
                                                   │  upload (GitHub Actions)
                                                   ▼
                    Databricks Volume /Volumes/house_prices/raw/landing
                                                   │  Auto Loader
                                                   ▼
            bronze.listings_raw ─> silver.* (parse, SCD2, lifecycle) ─> gold.* (Warsaw)
```

Databricks naming: catalog `house_prices`; schemas `raw` (volume `landing`), `bronze`, `silver`, `gold`.

## 3. Bronze contract

### 3.1 Envelope — one JSON line per record

| Field | Type | Example | Rule |
|---|---|---|---|
| `schema_version` | string | `"1.0"` | Bump on breaking change |
| `source` | string | `"olx"` | Lowercase source key |
| `adapter` | string | `"mock_api"` | Which code produced it |
| `adapter_version` | string | `"0.1.0"` | Package version or git SHA |
| `crawl_id` | string | `"olx-20261102-0600"` | Links to manifest |
| `fetched_at` | timestamp | `"2026-11-02T06:00:12Z"` | Always UTC |
| `record_type` | string | `"detail"` | `card`, `detail`, `development` |
| `source_listing_id` | string | `"1104369854"` | Site's own id, as text |
| `source_url` | string | URL | Tracking params removed |
| `fetch_status` | string | `"ok"` | `ok`, `not_found`, `gone`, `error` |
| `http_status` | int\|null | `200` | |
| `payload_format` | string | `"json"` | `json` or `html` |
| `payload` | string\|null | raw body | Untouched; null if status ≠ ok |
| `payload_sha256` | string\|null | hex | Cheap change detection |

Ship a JSON Schema at `src/house_prices/contract/schema/bronze_envelope.schema.json` and validate in tests.

### 3.2 Crawl manifest — one per crawl, written last

`crawl_id`, `source`, `started_at`, `finished_at`, `mode` (`full`|`incremental`), `scope` (e.g. `{"region": "PL", "type": "sale"}`), `as_of` (date or null), `cards_seen`, `details_fetched`, `errors`, `status` (`complete`|`partial`|`failed`).
Only a **complete full** crawl can prove a listing disappeared.

### 3.3 File layout

```
data/raw/{source}/ingest_date=YYYY-MM-DD/{crawl_id}/part-0001.jsonl.gz
data/raw/{source}/ingest_date=YYYY-MM-DD/{crawl_id}/_manifest.json
```
Same layout under the Volume. `ingest_date` = the simulated/business date (`as_of` or today).

### 3.4 Adapter rules

1. Append-only; never overwrite or delete in `raw/`.
2. No cleaning in adapters.
3. Duplicates allowed; removed downstream.
4. Removed listing re-checked ⇒ envelope with `fetch_status = gone`, null payload.
5. Re-running a day ⇒ new `crawl_id`.

## 4. Generator

### 4.1 World model
- **Property**: true attributes (location TERYT, type flat/house, area, rooms, floor, year built, true condition, plot area for houses, building type) and true sale price if sold. Never visible to the pipeline.
- **Listing**: one ad of a property on one source. A property can have several listings over time (reposts) and on both sources (cross-posts).
- **Development**: developer investment with sibling units (Otodom-like).
- **Truth export** to `truth/`: `properties`, `listing_property_map`, `events`, `renovation_labels`.

### 4.2 Determinism
World starts at `world_start`; day D is produced by replaying from day 0 with `seed`. No stored state needed. 20,000 properties must replay in well under a minute (vectorize / cache if needed).

### 4.3 Geography (all of Poland)
- Hierarchy: voivodeship → powiat → gmina → city/town/village → district/sub-district (large cities).
- Keys: TERYT codes (GUS). Warsaw powiat = `1465`. Ship `reference/locations_pl.csv` (small curated subset is fine to start: all 16 voivodeships, ~50 cities incl. all Warsaw districts, a sample of towns and villages).
- Distribution by population weight; Warsaw ≈ 10–15% of listings.
- Each location: base price per m² + monthly trend.
- Payload location is **text** at one of three precision levels (street + number, district, city only) + approximate coordinates (+ OLX map zoom 17 or 13). Text→TERYT is a silver task.

### 4.4 Daily events (configurable probabilities, not market facts)

| Event | Probability | Payload effect |
|---|---|---|
| New listing | 2–4% of active stock | New id |
| Price change | 3% (80% cuts of 1–10%) | Price, price/m² |
| Description edit | 2% | Text changed |
| Refresh / bump | 5% (OLX) | Only date line — **not** a real change |
| Field correction | 0.5% | e.g. area 84 → 84.5 |
| Sold | 1% | Disappears; true sale price 90–100% of last ask → truth |
| Withdrawn | 0.3% | Disappears |
| Repost | 0.3% | Old id gone; same property returns days later, new id, rewritten text |
| Cross-post | 20% of new properties | Also on the other source, different wording/fields |
| Temporary gap | 0.5% | Missing 1–2 days, returns unchanged |
| Promo (developer) | small | Price drop + `price_lowest_30d` |
| Price on request toggle | small | Number ↔ "Ask for price" |

### 4.5 Source renderers (English labels, Polish geography)

**OLX-like** (small fixed param set; detail lives in description)
- Flats: `price_per_m2` (string, decimal point), `floor` ("Ground floor", "1".."10", "Above 10"), `furnished` (Yes/No), `market` (Primary/Secondary), `building_type` (Block, Tenement, Apartment building), `area` ("117,56 m²" comma decimal), `rooms` ("2 rooms", "4 or more").
- Houses: `price_per_m2`, `market`, `area`, `plot_area` ("1 078 m²"), `building_type` (Detached, Terraced, Farmstead, Summer house), `floors` ("Single-storey", "One floor", "Single-storey with usable attic"). No rooms field.
- Price as formatted text ("999 000 zł") + `negotiable` flag; machine-readable integer price also present in a widget field.
- Date line in 3 formats: "Added 01 November 2026", "Refreshed today at 12:23", "Refreshed on 01 November 2026".
- Seller: type private/business, display name, profile id, member since; business adds company block (name, tax id, phone, email, address).
- Map: lat/lon + zoom (17 = street, 13 = city).
- No condition field.

**Otodom-like** (rich, sectioned)
- `characteristics` list with sections differing by type: flats (main, building & materials, equipment), houses (main, building details, plot & surroundings).
- Numeric price; price states: number | promo with `lowest_price_30d` | `price_on_request`.
- `floor` as "3/5"; multi-valued fields as arrays (heating, media, security, fencing, extras).
- `finishing_state`: ready to move in | to renovate | to finish | no information.
- "no information" sentinels distinct from real 0 (e.g. rent 0 vs no information).
- Advertiser type private | agency | developer; agency reference number on agency ads.
- `last_updated` date; JSON-LD block that occasionally disagrees with visible fields.
- Development pages: name, completion quarter, status, units available "3 of 6", area range, "from X zł/m²", unit table linking child ads.

### 4.6 Descriptions (English)
Template + phrase banks from true attributes, with noise: condition only in text ~40% of the time; contradictions (title vs field area/rooms, text location outside the location field); boilerplate ("Direct sale, no agents please"), capitals, emoji; bundled prices ("flat 555 000 + garage 85 000"); parking/storage priced separately; phone numbers in text (plain, "5-0-4-5…", "60*******69"); occasional typos in text prices/dates; cadastral parcel number ("plot 260/15") sometimes.

### 4.7 Renovation labels (truth + classifier target)
`needs_renovation`, `renovation_possible`, `renovated`, `to_finish` (+ `build_stage`: developer_standard | closed_shell), `unknown`.

### 4.8 Injected edge cases (each must have a test)
1. Whole-source outage one day — no listing marked sold.
2. Partial crawl (manifest `partial`, 30% missing) — same rule.
3. Duplicate envelope within one crawl.
4. Same listing fetched by two crawls in one day.
5. Price typo fixed next day (999 000 → 99 900 → back).
6. Refresh-only days create no history row.
7. Relist under new id 10 days after removal.
8. Listing reappears after being marked sold.
9. Late data: yesterday's file after today's.
10. Schema drift: new Otodom field from a set date.
11. Development with 5–10 near-identical sibling units on one street — must not merge.
12. Rent 0 vs "no information".
13. Parking/storage priced separately in text only.
14. Price ↔ "price on request" toggling.
15. Developer promo with 30-day lowest price, then reverts.
16. Development "units available" decreasing over time.
17. Same unit cross-posted by developer and agent, slightly different area and sub-district.
18. Area including garage on some listings of one development.
19. Field "ready to move in" vs text "needs refreshing".
20. "Available from" in the past on an active listing.
21. OLX date line switching formats for the same ad.
22. Area field holds built-up area; usable area only in text.
23. Rooms field disagrees with title.
24. Package price in text differs from field price.
25. Non-property listing (cabin for relocation) in houses.
26. Phone numbers in description (plain and obfuscated).
27. Location precision varies with map zoom.
28. Same seller profile reposts one property under a new id.

## 5. Mock listings API

| Endpoint | Returns |
|---|---|
| `GET /olx/api/v1/offers?category=flats|houses&region=&page=` | 50 cards: id, title, price text, date line, url |
| `GET /olx/api/v1/offers/{id}` | OLX-like detail; 404/410 after removal |
| `GET /otodom/api/v1/search?type=flat|house&region=&page=` | 50 cards |
| `GET /otodom/api/v1/ads/{id}` | Otodom-like detail |
| `GET /otodom/api/v1/investments/{id}` | Development page |
| `GET /health` | status + simulated date |

Behaviours (config `api:`): `X-API-Key` header required; rate limit (default 300/min) ⇒ 429 + `Retry-After`; ~1% 5xx and some slow responses; `as_of=YYYY-MM-DD` on every endpoint (used only for the initial 30-day backfill; default = today).

## 6. Adapters and runner

```python
class SourceAdapter(Protocol):
    source: str
    version: str
    def list_cards(self, scope: Scope, as_of: date | None) -> Iterator[RawRecord]: ...
    def fetch_detail(self, listing_id: str) -> RawRecord: ...
```
The **runner** (shared) handles envelopes, manifests, retries with backoff, rate limiting, gzip writing, and calls the adapter.
Crawl strategy: read all search pages daily (full), fetch details for new ids or cards whose price/date line changed, weekly full detail refresh, re-check vanished ids once (→ `gone`).
Templates in `src/house_prices/adapters/templates/`: `scraper_template.py`, `provider_api_template.py`, `csv_template.py` (examples only, not wired to real sites).

**Personal-use swap = 3 changes:** adapter, one `parse(payload) -> CanonicalListing` parser in `src/house_prices/parsers/`, and config (`sources`, `analysis_scope`).

## 7. Configuration

`config/generator.yaml`
```yaml
seed: 42
world_start: 2025-11-01
backfill_days: 30
properties: 20000
geography: {scope: PL, locations_file: reference/locations_pl.csv, weight_by: population}
api: {port: 8000, rate_limit_per_minute: 300, error_rate: 0.01}
truth_dir: truth/
```
`config/pipeline.yaml`
```yaml
sources:
  - {name: olx, adapter: mock_api, base_url: "http://localhost:8000/olx/api/v1"}
  - {name: otodom, adapter: mock_api, base_url: "http://localhost:8000/otodom/api/v1"}
raw_dir: data/raw
volume_path: /Volumes/house_prices/raw/landing
analysis_scope:
  - teryt: "1465"   # Warszawa
delisting_missed_full_crawls: 3
```

## 8. Silver

| Table | Grain | Purpose |
|---|---|---|
| `silver.listing_snapshot` | listing × crawl | Parsed, typed, deduped canonical fields |
| `silver.listing_history` | listing × version (SCD2) | `valid_from`, `valid_to`, `is_current`, `version`, `change_type` |
| `silver.listing_lifecycle` | listing | first_seen, last_seen, status (active/delisted/relisted), delisted_on, last_asking_price |
| `silver.crawl_health` | crawl | From manifests |
| `silver.development` | development × day | status, completion quarter, units available |
| `silver.dim_location` | location | TERYT hierarchy |

- **Canonical listing** fields (nullable where a source lacks them): source, listing_id, url, title, property_type, market, price_pln, price_on_request, is_promo, price_lowest_30d_pln, price_negotiable, area_m2, usable_area_m2, plot_area_m2, rooms, floor, building_floors, building_type, year_built, finishing_state, heating[], amenities[], monthly_rent_pln (+ rent_status), ownership, available_from, advertiser_type, agency_ref, investment_id, location_text, location_precision, teryt, lat, lon, description, description_sha256, refreshed_at, created_at, source_last_updated, seller_hash.
- **SCD2 tracked**: price_pln, price_on_request, area_m2, rooms, floor, finishing_state, description_sha256, title, advertiser_type. **Ignored**: refreshed_at, fetched_at, photos, field order.
- **Late data** merges into history by date, never becomes "current" wrongly.
- **Delisting**: missing from N (=3) consecutive **complete full** crawls ⇒ `delisted`; store `last_asking_price`, never call it a sale price; reappearance ⇒ `relisted`.
- **PII**: drop names/phones/emails; scrub phone patterns from description; hash seller/profile ids and NIP.

## 9. Gold (first versions, filtered by `analysis_scope`)
- `gold.price_history` — one row per listing per price version.
- `gold.comparables` — median price/m² by district × property type × area band × month.
- `gold.property_match` (later) — links reposts and cross-posts (keys: agency_ref, parcel number, area±1%, price, location, text similarity).

## 10. Notes for real sources
- OLX served plain HTTP in our test; Otodom blocked automated fetching (bot detection). Do not bypass bot protection, logins or CAPTCHAs.
- Real sources cannot backfill; history starts when you start crawling. Respect terms of use and robots.txt.
