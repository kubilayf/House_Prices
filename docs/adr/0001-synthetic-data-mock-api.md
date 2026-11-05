# 0001: Synthetic data served by a mock API

- **Status:** Proposed
- **Date:** 2026-11-05

> Reconstructed from `docs/spec.md` §1 and the team design docs. The spec records the decision and its reason, but not the alternatives, so the options below need the team's confirmation before this ADR is accepted.

## Context

The project needs a realistic, changing housing market to build a pipeline on, and the repo is public. Real listing sites have terms of use and may block automated access: Otodom blocked our automated fetching during research. Real data would also bring personal data into the project, and it could not be replayed for tests or for people who join later.

## Options

1. **Synthetic data behind a mock HTTP API:** legal, reproducible and publishable, and the adapter still deals with pagination, API keys, 429s and 5xxs. We have to build and maintain the simulator.
2. **Scraping real sites in the public repo:** real data, but it raises legal and personal-data problems, breaks when sites change, and cannot be reproduced.
3. **Synthetic files with no API:** the simplest option, but it skips the HTTP work a real source needs and leaves no realistic adapter to swap out.
4. **A paid data provider:** real data without scraping, but it costs money, and the data probably cannot be republished.

## Decision

Option 1. A seeded generator feeds a mock API with OLX-like and Otodom-like endpoints, and no scraping code for real sites ships in the public repo. Real sources plug in through adapters.

## Consequences

- Anyone can reproduce the same data from the seed, and tests can compare results against `truth/`.
- Swapping in a real source takes three changes: the adapter, one parser, and config. Those sources are the operator's responsibility, including terms of use and robots.txt.
- Event probabilities are configuration, not market facts, so results say nothing about the real market.
- The pipeline only meets the messiness we choose to simulate, which is why the spec lists injected edge cases.
