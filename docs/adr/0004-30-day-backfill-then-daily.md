# 0004: Load the last 30 days at the start, then daily

- **Status:** Proposed
- **Date:** 2026-11-05

> Reconstructed from `docs/spec.md` §1 and the team design docs. The spec records the decision and its reason, but not the alternatives, so the options below need the team's confirmation before this ADR is accepted.

## Context

The simulated world starts on `world_start` (2025-11-01), about a year ago. The pipeline needs enough history to show changes over time, and it runs on Databricks Free Edition, which has limited compute.

## Options

1. **Backfill 30 days through `as_of`, then crawl daily:** history is useful from day one and the load stays small. Older history is not loaded.
2. **Backfill everything since `world_start`:** the most history, but the highest risk of hitting Free Edition limits.
3. **Daily only, with no backfill:** the smallest load and the closest match to a real source, but it takes weeks before there is any history to work with.

## Decision

Option 1, with the window set by `backfill_days: 30`. `as_of` is used only for this initial backfill.

## Consequences

- The window can be extended later without code changes, by changing `backfill_days`.
- Listings created before the window first appear at its start, so `first_seen` in silver is not the true creation date.
- Real sources cannot backfill: their history starts on the first crawl. Nothing downstream may depend on `as_of` being available.
- **Open:** the Free Edition cost of the backfill and of a daily run has not been measured yet. The team plans to measure it and then decide how far back to extend.
