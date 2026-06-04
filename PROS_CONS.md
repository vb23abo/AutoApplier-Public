# Pros & Cons — Design Decisions

## Self-hostable monorepo

| Pros | Cons |
|------|------|
| No per-application SaaS fees | You operate Postgres, Redis, workers |
| Full data ownership | Setup friction vs Chrome extensions |
| Auditable application history | Security is your responsibility |

## Python + FastAPI backend

| Pros | Cons |
|------|------|
| Strong scraping/AI ecosystem | GIL for CPU-heavy work (mitigate with Celery) |
| Async SQLAlchemy + Alembic | Learning curve vs Django batteries |

## Playwright scrapers

| Pros | Cons |
|------|------|
| Real browser, handles SPAs | Slower than HTTP-only scrapers |
| Path to auto-submit (Phase 3) | DOM breakage, ToS/rate limits |

## Dual-repo (Public docs / Private code)

| Pros | Cons |
|------|------|
| Share progress without leaking IP | Two repos to maintain |
| Clear boundary for secrets | Manual sync of RESULTS/ROADMAP |

## Anthropic Claude (Phase 2+)

| Pros | Cons |
|------|------|
| Strong document generation | API cost per JD |
| Good instruction following | Requires key management in private repo |
