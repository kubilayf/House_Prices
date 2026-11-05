# 0002: Bronze contract with raw envelope and crawl manifest

- **Status:** Proposed
- **Date:** 2026-11-05

> Reconstructed from `docs/spec.md` §1 and the team design docs. The spec records the decision and its reason, but not the alternatives, so the options below need the team's confirmation before this ADR is accepted.

## Context

Sources differ in format (JSON or HTML) and in fields, and they will change over time. Everything downstream needs a stable way to read them, and must be able to tell "this listing is gone" apart from "our crawl failed".

## Options

1. **Raw envelope plus a crawl manifest, append-only:** every source lands the same way with its payload untouched, and parsing can be re-run later. Storage holds duplicates, and silver has to do all the parsing.
2. **Parse to a canonical schema during ingestion:** downstream is simpler, but a parser bug loses data for good, and every adapter has to contain cleaning logic.
3. **A separate raw format per source:** adapters are the easiest to write, but every downstream job needs per-source code.

## Decision

Option 1. Each record is one JSON line in the envelope from spec §3.1, and each crawl writes a manifest last (§3.2). Files are never overwritten or deleted, and re-running a day creates a new `crawl_id`.

## Consequences

- Any source works downstream without changes, and history can be rebuilt from bronze.
- Only a complete full crawl can prove a listing disappeared, so outages and partial crawls never mark anything as sold.
- Duplicates are allowed in bronze and must be removed downstream.
- Raw payloads may contain personal data, which silver has to drop or hash.
- Breaking changes require a `schema_version` bump and a spec update in the same PR.
- **Open:** `source_url` has tracking parameters removed, while adapters must do "no cleaning". The envelope should state that this rule applies to `payload` only, or the URL should stay raw.
