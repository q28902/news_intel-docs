# news_intel — Public Docs Mirror

Read-only documentation mirror of the private `news_intel` repository.

## What is this?

Public documentation for `news_intel` — a GDELT 2.0 dual-pipeline news intelligence infrastructure (collection, clustering, importance scoring, multilingual matching). The actual source code remains in a private repository.

This mirror exists so that architectural decisions, weekly task plans, and operational notes can be shared and discussed outside the private code context.

## Contents

- `SESSION_HANDOFF.md` — Authoritative project state (current version, phase, decisions)
- `week1_tasks.md`, `week2_tasks.md` — Weekly implementation plans (completed)
- `operations.md` — Operational playbook (launchd, health checks, notification helper)
- `decisions/` — Architectural decision records
- `observations/` — Empirical findings from production runs

## Sync policy

This mirror is updated **manually** when the private docs change materially. It may lag the private repo by hours or days. No code, no secrets, no personal identifiers.

## Not included

- Source code (`data/`, `bot/`, `shared/`, `scripts/`)
- Configuration (`.env`, credentials)
- Internal chat IDs, tokens, or absolute user paths (all sanitized to `<REPO_ROOT>` placeholders)

## License

Documentation is provided as-is for reference and discussion. See the private repo for code licensing.
