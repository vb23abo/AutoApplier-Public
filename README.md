# AutoApplier

> An intelligent, self-hostable job application agent — quality matching, full audit trail, no per-application SaaS fees.

**This repository is informational only.** Implementation code lives in the private `AutoApplier-Private` repository.

---

## Why AutoApplier?

- **Self-hostable** — run locally or on your own server
- **Quality over volume** — match scoring before auto-apply
- **Full audit trail** — snapshot of every resume and cover letter sent
- **Post-apply automation** — follow-ups, ghosting detection, reply parsing (roadmap)

---

## Tech stack (high level)

| Layer | Technology |
|-------|------------|
| Backend | Python 3.12, FastAPI |
| Frontend | Next.js 14, TypeScript |
| Automation | Playwright |
| Data | PostgreSQL 16, Redis, Celery |
| AI | Anthropic Claude (Phase 2+) |

---

## Module overview

| Module | Status | Description |
|--------|--------|-------------|
| Profile & Resume | Phase 1 | Profile store, PDF parsing |
| Job Discovery | Phase 1 | LinkedIn + Indeed scrapers, dedup |
| Match & Scoring | Phase 2 | Semantic fit, thresholds |
| Document Generation | Phase 2 | Tailored resume, cover letters |
| Form Filler | Phase 3 | ATS auto-submit |
| Application Tracker | Phase 3 | Lifecycle FSM, email parsing |
| Dashboard & Analytics | Phase 4 | UI, funnel, A/B metrics |

Details: [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)

---

## Roadmap

See [ROADMAP.md](ROADMAP.md) and [CHANGELOG.md](CHANGELOG.md).

**Current:** Phase 1 foundation — API, DB schema, scraper pipeline, minimal dashboard shell.

---

## Results & learnings

- [RESULTS.md](RESULTS.md) — benchmarks and outcomes (updated periodically)
- [PROS_CONS.md](PROS_CONS.md) — design tradeoffs and lessons

---

## Contributing

Ideas and feedback welcome via Issues. See [CONTRIBUTING.md](CONTRIBUTING.md).

**Note:** Source code is not published in this repo.

---

## Links

- GitHub: [vb23abo/AutoApplier-Public](https://github.com/vb23abo/AutoApplier-Public)
- License: MIT (see LICENSE)
