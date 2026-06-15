# JobCompass — Product Requirements Document (MVP)

**Document Type:** Product Requirements Document (PRD)
**Product:** JobCompass — AI-Powered Career & Job Search Platform
**Version:** 1.0 (MVP)
**Status:** Draft for Review
**Owner:** Product Management
**Audience:** Founders · Product · Design · Engineering · Stakeholders
**Last Updated:** June 15, 2026

---

## 1. Executive Summary

JobCompass is an AI-powered career platform built for tech professionals and recent graduates in the UK. It delivers an end-to-end job search experience — from resume diagnosis to intelligent job discovery, automated applications, and role-specific interview preparation — without depending on aggregators like LinkedIn or Indeed.

Its defining capability is a **direct-to-source job discovery engine** that scrapes company career websites and cross-references the UK Home Office's official sponsor licence register to surface graduate, sponsored, and non-sponsored roles alongside live visa sponsorship intelligence. Candidates know not just *where* to apply, but *whether* a company sponsors, *how actively*, and *what salary to expect* — before they spend a single minute on an application.

The MVP focuses on validating three core bets: that direct-source job discovery outperforms aggregator-dependent tools in accuracy and freshness; that resume-to-job fit scoring increases application quality; and that AutoApplier form-filling meaningfully reduces the time cost of applying.

| Attribute | Detail |
|---|---|
| **Product Name** | JobCompass |
| **Product Type** | AI-Powered Career & Job Search Platform |
| **Platform** | Responsive Web Application (Desktop + Mobile) |
| **Geography** | United Kingdom (MVP) |
| **Primary Users** | Tech professionals · Recent graduates (including international) |
| **MVP Target** | Resume analysis, job discovery (3 boards), AutoApplier, application tracker |
| **Primary Outcome** | Higher application quality + lower time-per-application |
| **North Star Metric** | Weekly Active Users who apply to ≥3 jobs via JobCompass and receive ≥1 response |

---

## 2. Product Vision

**Vision Statement:** *Every job seeker deserves a compass — not just a list of links.*

Job searching in the UK, particularly for international graduates and early-career tech professionals, is opaque, exhausting, and fragmented. Candidates spend hours on aggregators, guessing which companies sponsor visas, submitting generic resumes into black holes, and scrambling to prepare for interviews without knowing what to study.

JobCompass collapses this chaos into a single, intelligent workflow:

> **Resume → Discover → Match → Apply → Prepare → Track**

Every module feeds the next. A user's resume informs their job matches. Their matches inform their AutoApplier. Their applications inform their interview prep. The platform gets more useful the longer they use it.

**Long-term platform ambition:** JobCompass starts as a UK job search tool but its architecture — direct scraping, live sponsorship data, AI fit scoring — is geography-agnostic. V2 expands to EU and Canada. V3 introduces employer-side tooling. The enduring promise is *navigational clarity*: at every stage of a job search, a user knows exactly where they stand and what to do next.

**Strategic Positioning:** JobCompass sits above free aggregator tools (stale data, no intelligence, ad-heavy) and below expensive human recruiters (slow, inaccessible). It offers direct-source freshness, sponsorship intelligence, and AI-driven personalisation at a self-serve price point — a combination no existing product offers.

---

## 3. Background and Context

### 3.1 The UK Graduate Job Market

The UK graduate jobs market is intensely competitive. Universities produce hundreds of thousands of graduates annually who compete for a limited pool of structured graduate schemes and entry-level tech roles. International students face an additional filter: the requirement for employer visa sponsorship under the Skilled Worker route, which eliminates a large proportion of available roles from their consideration.

Despite this complexity, most candidates search for jobs on generalist aggregators — platforms that mix sponsored and non-sponsored listings without clear labelling, show stale postings, and offer no signal on how actively a company hires international candidates.

### 3.2 The Sponsorship Intelligence Gap

The UK Home Office publishes and regularly updates a public register of licensed sponsors — companies legally permitted to sponsor Skilled Worker and Graduate visas. This data is publicly accessible but no consumer job platform meaningfully integrates it. Candidates either don't know it exists, or manually cross-reference it one company at a time.

There is additional sponsorship signal available: immigration transparency reports, Companies House data, and web scraping of employer profiles can approximate *how many* international hires a company makes, giving candidates a proxy for cultural willingness to sponsor — not just legal ability.

### 3.3 The ATS and Resume Problem

Applicant Tracking Systems (ATS) are used by a large majority of UK employers of significant size. Resumes that are poorly structured, keyword-light, or formatted for human eyes over machine parsing are silently filtered out. Candidates receive no feedback. Most early-career applicants have never had a professional resume review.

### 3.4 The Aggregator Dependency Problem

LinkedIn, Indeed, and similar platforms are intermediaries — companies post to them, and candidates search them. This creates freshness lag (roles appear hours or days after posting), data quality issues (expired roles remaining live), and information loss (salary, sponsorship status, actual application link often stripped). Going direct to company career sites eliminates all three problems.

### 3.5 Market Tailwinds

- Rising international student enrollment in UK universities (increasing demand for sponsorship intelligence)
- Growth in remote and hybrid roles (increasing the number of companies worth applying to)
- Widespread ATS adoption (increasing penalty for resume quality issues)
- Maturing LLM capabilities (making AI-driven resume analysis and form-filling viable at low marginal cost)

---

## 4. Problem Statement

Job seekers in the UK — particularly tech graduates and international candidates — face five compounding problems:

1. **Discovery friction:** Job listings are scattered across hundreds of company career sites, making direct-source search impractical without tooling.

2. **Sponsorship opacity:** Candidates cannot quickly determine whether a company sponsors visas, let alone how actively or at what salary bands.

3. **Resume blindness:** Most candidates submit resumes without knowing whether they are ATS-compatible, keyword-relevant, or appropriately formatted for their experience level.

4. **Application volume without quality:** Without intelligent matching, candidates apply broadly and get low response rates — wasting time and missing better-fit roles.

5. **Preparation gaps:** Candidates who reach interviews often under-prepare for role-specific technical and behavioural questions, reducing conversion at the final stage.

JobCompass addresses all five — sequentially, within a single platform.

---

## 5. Target Users

### Persona 1 — The International Tech Graduate

**Name:** Priya, 23
**Background:** MSc Computer Science, Russell Group university. Graduating in 6 months. Needs a Skilled Worker visa sponsor to remain in the UK.
**Key frustrations:** Cannot quickly filter to sponsored roles. Doesn't know which companies actively sponsor. Has a decent resume but has never had it reviewed. Anxious about wasting limited application energy on ineligible companies.
**What success looks like:** Applies only to roles where she is likely eligible, with a tailored resume, in less time than her current manual process.

---

### Persona 2 — The Early-Career Tech Professional

**Name:** James, 26
**Background:** 2 years experience as a junior developer. UK citizen. Looking to move into a mid-level role or a structured graduate scheme at a larger company.
**Key frustrations:** Applies to many roles, gets few responses. Doesn't know if his resume is the problem or the role targeting. Spends hours a week manually applying to roles on multiple platforms.
**What success looks like:** Gets a clearer picture of why he's being rejected, applies to better-fit roles with a stronger resume, and reduces time spent on applications from hours to minutes.

---

## 6. Module Definitions — Inputs, Processing, Outputs

### Module 1 — Authentication & User System

**What it does:** Manages user identity, session, profile persistence, and UI preferences.

| | Detail |
|---|---|
| **Input** | Email + password / OAuth (Google, GitHub) |
| **Processing** | JWT auth, bcrypt hashing, session management |
| **Output** | Authenticated session, persistent user profile, role-based access |
| **UI Settings** | Light / dark / system theme, accent colour preferences |

**Key design note:** Profile setup is a guided wizard, not a form dump. It collects name, current role, years of experience, target role, target salary band, work eligibility (UK citizen / visa required), and preferred UK cities. This data seeds every downstream module.

---

### Module 2 — Resume Dashboard

**What it does:** Accepts a resume upload, extracts structured profile data, identifies gaps, scores ATS compatibility, and returns prioritised improvement recommendations.

| | Detail |
|---|---|
| **Input** | Resume file (PDF or DOCX), user profile (from Module 1) |
| **Processing** | Text extraction → section parsing → skills taxonomy mapping → ATS simulation → gap analysis via LLM → scoring model |
| **Output** | ATS score (0–100), section-by-section breakdown, gap report (skills present in profile but missing from resume), prioritised improvement tips, formatting recommendations |

**Scoring dimensions:**
- **Parsability** — can an ATS correctly extract name, contact, education, experience, skills?
- **Keyword coverage** — how many high-value keywords for the user's target role are present?
- **Content quality** — are bullet points action-verb led? Quantified where possible? Passive language flagged?
- **Formatting** — single page for 0–2 years experience, two pages for 3–8 years, correct font sizes, no tables or graphics that break parsers
- **Completeness** — are all expected sections present (summary, skills, experience, education, projects)?

**Resume → Profile bridge:** Upon first upload, the system extracts structured data (skills, experience, education, tech stack) and pre-populates the user profile. Subsequent uploads compare against the existing profile to highlight regressions or improvements.

---

### Module 3 — Job Discovery Engine (Core Infrastructure)

**What it does:** Runs a continuous scraping loop across company career websites to build and maintain a fresh, structured job database — *without relying on LinkedIn, Indeed, or any third-party aggregator.*

| | Detail |
|---|---|
| **Input** | Target company list (sourced from UK sponsor register + curated tech employer database), scrape schedule, field extraction config |
| **Processing** | Playwright-based headless browser → career page detection → job listing extraction → deduplication → structured normalisation → sponsorship tagging |
| **Output** | Structured job records in database: title, company, location, posted date, contract type, salary band (extracted or estimated), direct application URL, sponsorship status, sponsorship history score |

**Sponsorship data pipeline:**
- Primary source: [Home Office Register of Licensed Sponsors](https://www.gov.uk/government/publications/register-of-licensed-sponsors-workers) — updated weekly, parsed and stored
- Enrichment: CoSoC (Certificate of Sponsorship) count estimation from immigration transparency data and historical scraping
- Output fields per job: `is_sponsorable: bool`, `sponsor_6m_count: int`, `sponsor_12m_count: int`

**Salary intelligence:**
- Extracted from job posting where present
- Where absent: estimated using role title + seniority + company size + location via LLM inference against market data
- Output: `salary_min`, `salary_max`, `salary_confidence: low | medium | high`

**Job schema (per record):**
```
job_id          UUID
title           string
company         string
location        string
distance_km     float (from user's preferred city, computed at query time)
posted_at       datetime
contract_type   full-time | part-time | contract | graduate-scheme
is_sponsorable  bool
sponsor_6m      int
sponsor_12m     int
salary_min      int
salary_max      int
salary_currency GBP
apply_url       string (direct link to company career page)
source_domain   string
scraped_at      datetime
is_active       bool
```

---

### Module 4 — Graduate Roles Dashboard

**What it does:** Presents a filtered, paginated view of graduate scheme and entry-level roles discovered by the scraping engine.

| | Detail |
|---|---|
| **Input** | User profile (target role, location, work eligibility), filter selections |
| **Filters** | Time posted (24h / 48h / 72h / 1 week / any), distance from city (10 / 25 / 50 / 100 miles / nationwide), sponsorship required (yes / no / show all), salary band |
| **Processing** | Query job DB → apply filters → compute resume match score per result → rank |
| **Output** | Paginated job table with match score, sponsorship status, salary estimate, direct apply link |

**Table columns:** Job Title · Company · Location · Distance · Posted · Sponsorable · Sponsor Activity · Salary · Match % · Apply

---

### Module 5 — Sponsored Roles Dashboard

**What it does:** Presents roles specifically at companies on the Home Office sponsor register, prioritised by sponsorship activity.

Identical filter and table architecture to Module 4, with additional:
- **Sponsor tier filter:** Active (10+ CoS in 12 months) / Moderate (3–9) / Low (1–2) / Registered only (0 recent)
- **Visa route filter:** Skilled Worker / Graduate Route / Both

---

### Module 6 — Non-Sponsored Roles Dashboard

**What it does:** Presents roles at companies not on the sponsor register — relevant for UK citizens and settled status holders.

Identical filter and table architecture. Sponsorship columns hidden. Replaces sponsor filter with:
- **Company size filter:** Startup (1–50) / SME (51–250) / Mid-market (251–1000) / Enterprise (1000+)
- **Remote filter:** On-site / Hybrid / Remote / Any

---

### Module 7 — Resume-to-Job Match Engine

**What it does:** At query time, computes a fit score between the user's active resume and each job in the result set.

| | Detail |
|---|---|
| **Input** | User's parsed resume (skills, experience, seniority), job record (title, description, required skills) |
| **Processing** | Embedding-based semantic similarity + keyword overlap scoring + seniority fit check |
| **Output** | `match_score` (0–100) per job, top 3 matching reasons, top 3 gap reasons |

This score surfaces in the job table as a **Match %** column and drives default sort order.

---

### Module 8 — AutoApplier

**What it does:** Automatically fills and submits job application forms on company career sites on behalf of the user — using their profile, resume, and AI-generated responses to screening questions.

| | Detail |
|---|---|
| **Input** | Selected job(s) from dashboard, user profile, active resume, cover letter preferences |
| **Processing** | Playwright browser agent → form detection → field mapping → AI answer generation for screening Qs → human-timing simulation → file upload → submission |
| **Output** | Submission confirmation, application record created in tracker, document snapshot saved (exact resume + cover letter version sent) |

**Supported ATS platforms (MVP):** Direct career forms (custom), Greenhouse, Lever, Workday Basic
**CAPTCHA handling:** Detect → pause → notify user → resume on manual resolution
**Human-timing simulation:** Randomised keystroke delays (80–200ms), mouse movement patterns, randomised start time within a ±30 minute window of user-configured schedule
**Daily application cap:** User-configurable (default: 10/day). Exceeding cap queues for next day.
**Quality gate:** AutoApplier only fires for roles above user-configured match threshold (default: 65%).

---

### Module 9 — Application Tracker

**What it does:** Maintains a complete, structured log of every application — whether submitted via AutoApplier or manually logged by the user.

| | Detail |
|---|---|
| **Input** | Application records (from AutoApplier or manual entry), email parsing (IMAP / Gmail API) for reply detection |
| **Processing** | Status FSM: `saved → applied → viewed → screening → interview → offer → rejected → withdrawn` |
| **Output** | Kanban and table views, status timeline per application, document snapshot viewer, follow-up draft generator |

**Auto-status updates:** Email parser detects recruiter replies and updates status automatically.
**Follow-up logic:** At day 7 and day 14 post-application with no response, draft a follow-up email using Claude and surface it for user approval.
**Ghosting detection:** Mark as `no_response` after 21 days with no email activity.

---

### Module 10 — Interview Preparation

**What it does:** Generates a role-specific interview prep guide based on the job description, company, and user's profile gaps.

| | Detail |
|---|---|
| **Input** | Job record (title, JD, company), user profile (current skills, experience gaps identified in Module 2) |
| **Processing** | LLM analysis of JD → likely interview format (technical / behavioural / case) → topic extraction → resource curation |
| **Output** | Structured prep guide: likely question areas, recommended topics to study, curated resources (YouTube channels, official docs, blog posts), estimated prep time per topic |

**Resource curation logic:**
- For languages/frameworks (Python, Java, React): official docs + curated YouTube channels (e.g. Neetcode for algorithms, Traversy Media for web)
- For behavioural: STAR method templates pre-filled with user's experience from profile
- For system design: architecture resource links (e.g. ByteByteGo) scoped to the seniority level of the role

---

## 7. Technology Recommendations

### 7.1 Why these choices beat the competition

The market alternatives (LazyApply, Sonara, JobCopilot) share a critical architectural weakness: **they depend on LinkedIn and Indeed as their data source.** This makes them subject to rate limiting, ToS changes, data staleness, and information loss (salary, sponsorship, direct apply URL stripped by the aggregator). JobCompass's direct-to-source architecture is defensible in a way aggregator-dependent tools are not.

### 7.2 Recommended Stack

| Layer | Technology | Rationale |
|---|---|---|
| **Backend** | Python 3.12 + FastAPI | Best ecosystem for scraping, AI SDKs, async queues, and data pipelines |
| **Frontend** | Next.js 14 + TypeScript | App Router, RSC for speed, Tailwind + Shadcn/ui for UI |
| **Browser Automation** | Playwright + stealth plugin | Reliable ATS form detection; stealth reduces bot detection |
| **Database** | PostgreSQL 16 | Relational integrity for job/user/application data; JSONB for flexible JD fields |
| **Search / Filtering** | PostgreSQL full-text + pgvector | Hybrid keyword + semantic search without a separate search service in MVP |
| **Queue / Cache** | Redis + Celery | Async scrape jobs, AutoApplier task queue, result caching |
| **AI / LLM** | Anthropic Claude (claude-sonnet-4-6) | Resume analysis, cover letter generation, screening Q answers, interview prep |
| **Embeddings** | Anthropic Embeddings or OpenAI text-embedding-3-small | Resume↔JD semantic match scoring |
| **Email Parsing** | Gmail API + IMAP | Application status auto-update from recruiter replies |
| **Auth** | Supabase Auth or Auth.js | OAuth (Google, GitHub) + email/password; JWT sessions |
| **File Storage** | S3-compatible (Cloudflare R2) | Resume uploads + application document snapshots |
| **Deployment** | Docker Compose (dev) → Railway or Render (prod MVP) | Fast iteration, low ops overhead for MVP |
| **Gov.uk Data** | Scheduled HTTP fetch + CSV parse | Home Office sponsor register updated weekly; no API needed |

### 7.3 Technologies to evaluate for competitive edge

| Technology | Use case | Potential edge |
|---|---|---|
| **Browserless.io or Bright Data** | Scraping at scale without IP blocks | Sustains scrape volume as company list grows to thousands |
| **Apache Airflow** | Scheduling scrape jobs across thousands of company sites | Replaces Celery beat for complex DAG-based schedules at scale |
| **Qdrant or Weaviate** | Dedicated vector store for semantic job matching | Outperforms pgvector at scale (>1M job records) |
| **Resend or Postmark** | Transactional email (digests, follow-up drafts) | More reliable than SendGrid for deliverability |
| **Posthog** | Product analytics (funnel, session replay) | Understand where users drop off in the Resume → Apply flow |

---

## 8. What is Out of Scope for MVP

The following are deliberately excluded to ensure the MVP ships within 4 weeks and validates the core hypothesis before expanding:

| Feature | Reason deferred |
|---|---|
| Resume builder (create from scratch) | Requires separate editor UI; analyse-first approach is faster to validate |
| Cover letter module | Ships in V1.1 as part of AutoApplier enhancement |
| Employer / recruiter-side tooling | Different GTM; separate product consideration |
| EU / global job markets | Architecture supports it; geographic expansion is V2 |
| Mobile native app (iOS / Android) | Responsive web covers MVP; native is V2 |
| LinkedIn profile analysis | Out of scope for V1; resume-first is the right sequencing |
| Salary negotiation module | Post-offer tooling; deferred to V2 |
| Referral / networking automation | Ethical complexity; separate product consideration |

---

## 9. Success Metrics

### North Star
**Weekly Active Users who apply to ≥3 jobs via JobCompass and receive ≥1 recruiter response**

### Module-Level Metrics

| Module | Key Metric |
|---|---|
| Resume Dashboard | % of users who re-upload after seeing their first score (improvement intent) |
| Job Discovery | Job freshness: % of listings posted within 48h of appearing in JobCompass |
| Match Engine | Resume match score correlation with actual recruiter response rate |
| AutoApplier | Successful form submission rate (target: >90% of attempts) |
| Application Tracker | % of applications with auto-detected status update (email parsing success) |
| Interview Prep | User-reported confidence score before/after using prep guide |

### Launch Hypothesis
> If a candidate uses the Resume Dashboard + Job Discovery + AutoApplier, their application-to-response rate will be meaningfully higher than their historical rate on aggregator platforms.

---

## 10. Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Company career sites block scraper | High | High | Stealth Playwright, rotating user agents, Browserless.io fallback, respectful crawl delays |
| ATS platforms change form structure | Medium | High | Modular form-detector design; fallback to "open in browser" notification |
| Home Office sponsor register format changes | Low | Medium | Scheduled validator alert; manual parse fallback |
| LLM latency on resume analysis (>10s) | Medium | Medium | Async processing with progress indicator; stream partial results |
| CAPTCHA on application forms | High | Low | Detect → pause → notify user workflow; never block silently |
| Low scrape coverage for small company sites | Medium | Medium | Supplement with curated manual company list for UK tech employers |

---

## 11. Open Questions

1. **Pricing model:** Free tier (limited analyses + applications/month) vs. flat subscription vs. pay-per-application? What is the conversion threshold?
2. **Scrape legality:** Are terms of service for company career sites a concern at MVP scale? Should we implement a robots.txt checker per domain?
3. **Resume storage:** Should we store user resumes indefinitely or enforce a retention window (e.g. 90 days)?
4. **Sponsorship count accuracy:** CoS count estimates from public data will be imprecise. How do we communicate confidence levels without misleading users?
5. **Application volume ethics:** Should there be a platform-wide cap per company per day to avoid overwhelming small employers?
