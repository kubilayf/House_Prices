# 0003: Generate all of Poland, filter to Warsaw only in gold

- **Status:** Proposed
- **Date:** 2026-11-05

> Reconstructed from `docs/spec.md` §1 and the team design docs. The spec records the decision and its reason, but not the alternatives, so the options below need the team's confirmation before this ADR is accepted.

## Context

The team's analysis is about Warsaw, but the design should extend to other cities. Real ads also span many regions and include villages, and some ads give a location that conflicts with their text.

## Options

1. **Generate all of Poland and filter in gold through `analysis_scope`:** a new city is a config change. Every layer below gold processes more data.
2. **Generate only Warsaw:** the smallest data volume, but adding a city means regenerating data and changing code, and out-of-city cases disappear.
3. **Generate all of Poland and filter in silver:** less silver processing, but widening the scope later means reprocessing silver as well.

## Decision

Option 1. Bronze and silver hold all of Poland. Gold joins `analysis_scope` to `dim_location` on TERYT codes (Warsaw powiat is `1465`).

## Consequences

- Adding a city or district takes one config line and a gold rebuild.
- Warsaw is only about 10–15% of listings, so most data processed in silver is outside the current analysis. This uses Free Edition capacity (see ADR 0004).
- Mapping location text to TERYT becomes a required silver task, because payloads carry location as text.
- TERYT codes must be handled as text, because some have leading zeros.
