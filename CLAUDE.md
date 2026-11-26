# holo_snowman

Projection-mapping software for a physical snowman statue (three balls + flat hat/hair panel + two arm panels). Users draw on and place stickers onto the snowman; the result is projected onto the statue.

## Agent skills

### Issue tracker

Issues live in GitHub Issues (SuperPing44/holo_snowman), managed via the `gh` CLI. See `docs/agents/issue-tracker.md`.

### Triage labels

Default vocabulary: needs-triage, needs-info, ready-for-agent, ready-for-human, wontfix. See `docs/agents/triage-labels.md`.

### Domain docs

Single-context: one `GLOSSARY.md` + `docs/adr/` at the repo root. See `docs/agents/domain.md`.

## Git workflow

- `main`: only stable, working states. Updated only by merging `dev` when something stands.
- `dev`: integration branch. All feature work merges here.
- Feature branches: branch off `dev` (e.g. `feature/<short-name>`), merge back into `dev`.
- Never commit feature work directly to `main` or `dev`.
- Exception: `research/*` branches from wayfinder research tickets are throwaway and never merged.
