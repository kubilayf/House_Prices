# 0007: Handbook and docs in GitHub, chat in Discord

- **Status:** Proposed
- **Date:** 2026-11-05

> Reconstructed from `docs/spec.md` §1 and the team design docs. The spec records the decision and its reason, but not the alternatives, so the options below need the team's confirmation before this ADR is accepted.

## Context

The team has 3+ contributors, and people will join later. Late joiners need to find out what was decided and why, without reading through old chat.

## Options

1. **Handbook and docs in GitHub, chat in Discord:** durable knowledge is versioned and reviewed next to the code, and day-to-day talk stays quick. Decisions made in chat have to be written down deliberately.
2. **Discord for everything:** the least effort, but decisions get lost in the scrollback.
3. **A separate wiki or docs tool:** easy to edit, but it sits away from the code and PR review, and it may add accounts or cost.

## Decision

Option 1. Anything people need to find later lives in GitHub, and Discord is for conversation.

## Consequences

- Late joiners have a searchable history that doesn't depend on chat.
- Docs changes go through PRs, which adds some friction.
- A decision reached in Discord doesn't count until it is recorded in GitHub. Hard-to-reverse decisions need an ADR.
- **Open:** the Data Lab design doc puts ADRs in a shared `handbook` repository and direction debates in GitHub Discussions, while this repo keeps ADRs in `docs/adr/`. The team should agree where project-level ADRs live.
