# 1. Record architecture decisions

Date: 2026-10-16

## Status

Accepted

## Context

modelship makes significant, consequential choices: serving stack, rollout
orchestration engine, admission control policy, registry strategy. These
choices have long lifetimes and will be questioned — by future contributors,
in interviews, and by the author six months from now.

## Decision

We record every significant architectural choice as an Architecture Decision
Record (ADR), using the template in [template.md](template.md).

Rules:

- ADRs are numbered sequentially.
- ADRs are immutable once accepted; superseding a decision means writing a
  new ADR that references the old one.
- One decision per ADR.
- PRs that implement a decision should link the ADR.

## Consequences

- Every design choice in this repo has a written justification.
- Debates are settled in text, not memory.
- The ADR list doubles as a guided tour of the architecture.
