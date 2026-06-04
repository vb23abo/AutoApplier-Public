# Roadmap

## Phase 1 — Foundation (in progress)

- [x] Monorepo structure (private repo)
- [x] Docker Compose: Postgres + Redis
- [x] DB schema: users, profiles, jobs, applications, documents
- [x] FastAPI: JWT auth, health, profiles, jobs API
- [x] Scrapers: LinkedIn + Indeed pipeline with dedup
- [x] Resume PDF parser
- [x] Next.js dashboard shell
- [ ] Production-hardened scraper selectors
- [ ] 100+ jobs/day stored reliably

## Phase 2 — Match & Documents

- [ ] JD parser (role, skills, salary, seniority)
- [ ] Semantic match scoring
- [ ] Per-JD resume tailoring
- [ ] Cover letter generator (tone variants)
- [ ] Screening question answerer

## Phase 3 — Auto-Submit & Tracker

- [ ] Playwright Easy Apply + Greenhouse/Lever
- [ ] CAPTCHA pause + notify
- [ ] Application FSM + email reply parsing

## Phase 4 — Dashboard & Analytics

- [ ] Full UI: profile wizard, queue review, rules
- [ ] Funnel and A/B analytics
- [ ] Follow-up drafts, audit log
