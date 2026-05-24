# AutoApplier 

> An intelligent, open-source job application agent that finds, tailors, and submits job applications on your behalf — with full auditability and no per-application fees.

---

## Why Autoapplier?

Getting a job in 2026 takes an average of 32–42 applications per interview. Every competitor in this space is a closed SaaS with per-application pricing, volume-first thinking, and no transparency into what was sent where.

Autoapplier is different:

- **Self-hostable** — run it locally or on your own server, no subscriptions
- **Quality over volume** — AI match scoring rejects weak fits before applying
- **Full audit trail** — every application stores a snapshot of the exact resume and cover letter sent
- **Post-apply automation** — follow-up drafts, ghosting detection, reply parsing
- **Senior role support** — a "quality mode" with deep JD analysis and human-review gate

---

## Modules

### Core
| Module | Description |
|---|---|
| **Profile & Resume Engine** | Structured user profile store, resume parser (PDF/DOCX), skills taxonomy, multiple resume variants per role type, per-JD ATS keyword injection |
| **Job Discovery & Scraper** | Multi-board aggregator: LinkedIn, Indeed, Greenhouse, Lever, Workday. Dedup pipeline, freshness filter, niche board support (AngelList, Otta, Wellfound) |

### Intelligence
| Module | Description |
|---|---|
| **Match & Scoring Engine** | JD ↔ profile semantic similarity, salary band check, role seniority fit, scam/spam detector, configurable match threshold before auto-apply fires |
| **Document Generation** | Per-JD tailored resume (keyword swap, bullet reranking), cover letter generation with tone options, screening question answerer |

### Automation
| Module | Description |
|---|---|
| **Form Filler & Submitter** | Playwright-based browser agent, ATS form detection (Workday, Lever, Greenhouse, iCIMS), human-timing simulation, CAPTCHA pause + notify, file upload handler |
| **Application Queue & Rules** | Daily/weekly caps, company blacklist/whitelist, manual review queue for borderline matches, retry logic for failed submissions |

### Tracking
| Module | Description |
|---|---|
| **Application Tracker** | Full lifecycle FSM: applied → viewed → screening → interview → offer. Email parser for auto status updates, calendar integration |
| **Notification & Follow-up** | Status change alerts (email/push), auto-draft follow-up emails at configurable intervals, ghosted application detection, daily digest |

### Analytics & Platform
| Module | Description |
|---|---|
| **Analytics Dashboard** | Response rate by board/role/resume variant, A/B testing cover letter styles, best-performing keywords, time-to-reply heatmap, funnel visualisation |
| **User Dashboard & Config** | Web UI for profile setup, rule config, queue review, and analytics. OAuth auth, API key management, full audit log |

---

## Competitive Landscape

| Tool | Strengths | AI Matching | Full ATS Support | Open/Self-host |
|---|---|:---:|:---:|:---:|
| LazyApply | Volume, Chrome extension, LinkedIn/Indeed | ⚠️ | ⚠️ | ❌ |
| Sonara | Quality matching, thoughtful outreach | ✅ | ⚠️ | ❌ |
| JobCopilot | Niche boards, learning feedback loop | ✅ | ⚠️ | ❌ |
| ApplyGenie | Scoring per posting, portfolio links | ✅ | ⚠️ | ❌ |
| LoopCV | Always-on campaigns, EU-friendly | ⚠️ | ⚠️ | ❌ |
| scale.jobs | Human experts, 93% hired in 90 days | ✅ | ✅ | ❌ |
| **Autoapplier** | **All of the above + self-hosted + open** | ✅ | ✅ | ✅ |

### Market gaps this project fills

1. **True self-hostable, open-source core** — no per-application fees, no SaaS lock-in
2. **Multi-resume A/B testing** — route resume variants by job type and measure response rates scientifically
3. **Post-apply pipeline automation** — follow-ups, ghosting detection, reply parsing, tracker state updates
4. **Senior/niche role support** — deep JD analysis, custom narrative, human-review gate for high-value roles
5. **Deep Workday/Greenhouse/iCIMS support** — most bots only handle Easy Apply; full ATS form support is genuinely rare
6. **Explainability & audit log** — document snapshot per application, every action logged

---

## Tech Stack

| Layer | Choice | Why |
|---|---|---|
| Backend | Python 3.12 + FastAPI | Best ecosystem for scraping, AI SDKs, and async queues |
| Frontend | Next.js 14 + TypeScript | Fast dashboard, RSC, Tailwind, Shadcn/ui |
| Browser automation | Playwright | Reliable cross-browser, supports stealth plugins |
| Database | PostgreSQL 16 | Relational, audit-friendly, JSONB for flexible JD data |
| Queue / cache | Redis + Celery | Async job runner for scraping and submission tasks |
| AI | Anthropic Claude (claude-sonnet-4-6) | Document generation, match scoring, Q&A answering |
| Infra | Docker Compose | Local-first, easy self-host |

---

## Project Structure

```
autoapplier/
├── backend/
│   ├── app/
│   │   ├── api/          # FastAPI routers
│   │   ├── models/       # SQLAlchemy models
│   │   ├── services/
│   │   │   ├── scraper/      # Job board scrapers
│   │   │   ├── matcher/      # Scoring engine
│   │   │   ├── generator/    # Resume & cover letter gen
│   │   │   ├── submitter/    # Playwright form filler
│   │   │   └── tracker/      # Application lifecycle
│   │   └── tasks/        # Celery tasks
│   ├── pyproject.toml
│   └── alembic/          # DB migrations
├── frontend/
│   ├── app/              # Next.js App Router
│   ├── components/
│   └── package.json
├── infra/
│   └── docker-compose.yml
└── docs/
    └── ARCHITECTURE.md
```

---

## Roadmap —

### Foundation & Scraper
- [ ] Repo setup, monorepo structure, Docker Compose (Postgres + Redis)
- [ ] DB schema: users, profiles, jobs, applications, documents
- [ ] FastAPI skeleton with JWT auth, health check, basic routing
- [ ] Job scraper for LinkedIn Easy Apply + Indeed (Playwright)
- [ ] Dedup pipeline + job freshness filtering
- [ ] Profile CRUD API + resume PDF parser (pdfplumber)

**Milestone:** scrape 100+ jobs/day and store them

---

### Match Engine & Document Generation
- [ ] JD parser — extract role, skills, salary, seniority from raw HTML
- [ ] Semantic match scoring (embeddings via Anthropic)
- [ ] Per-JD resume tailoring: keyword injection, bullet reranking
- [ ] Cover letter generator with 3 tone variants (formal / conversational / concise)
- [ ] Screening question answerer using profile data + Claude
- [ ] Application queue with match threshold config

**Milestone:** given a JD, generate a tailored resume + cover letter in under 5 seconds

---

### Auto-Submit & Tracker
- [ ] Playwright agent: LinkedIn Easy Apply end-to-end
- [ ] Greenhouse + Lever form filling
- [ ] Human-timing simulation: randomised delays, mouse movement patterns
- [ ] CAPTCHA detection → pause + push notification
- [ ] Application tracker DB + status FSM
- [ ] Email parser for reply detection (IMAP / Gmail API)

**Milestone:** apply to 20+ jobs/day autonomously end-to-end

---

### Dashboard, Analytics & Polish
- [ ] Next.js dashboard: profile setup wizard, queue review, rule config
- [ ] Analytics views: funnel, response rate by board/variant, keyword performance
- [ ] Follow-up email drafter (day 7 / day 14, Claude-generated)
- [ ] Notification system: email digest + push alerts
- [ ] Audit log with document snapshot per application
- [ ] Rate limiting, error handling, retry hardening

**Milestone:** working MVP with UI, fully deployable via Docker

---

## Local Development Setup

### Prerequisites

Install these before writing any product code:

```bash
# Python 3.12
brew install python@3.12

# Node.js 20 LTS
brew install node

# GitHub CLI
brew install gh && gh auth login

# uv (fast Python package manager)
pip install uv

# Playwright + browsers
pip install playwright && playwright install

# Docker Desktop
# → https://www.docker.com/get-started
```

### Getting started

```bash
# Clone the repo
git clone https://github.com/YOUR_USERNAME/autoapplier.git
cd autoapplier

# Start Postgres + Redis
docker compose -f infra/docker-compose.yml up -d

# Backend
cd backend
uv sync
uv run alembic upgrade head
uv run uvicorn app.main:app --reload

# Frontend (separate terminal)
cd frontend
npm install
npm run dev
```

### Environment variables

Copy `.env.example` to `.env` and fill in:

```env
DATABASE_URL=postgresql://postgres:local@localhost:5432/autoapplier
REDIS_URL=redis://localhost:6379
ANTHROPIC_API_KEY=your_key_here
```

---

## Branch Strategy

```
main          ← stable, tagged releases only
└── develop   ← integration branch
    └── feature/scraper
    └── feature/match-engine
    └── feature/form-filler
    └── feature/tracker
    └── feature/dashboard
```

---

## Status

🟡 **In active development**

Started: May 2026.
