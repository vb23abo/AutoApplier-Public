# JobCompass — Technical Requirements Document (TRD)

**Document Type:** Technical Requirements Document
**Product:** JobCompass — AI-Powered Career & Job Search Platform
**Version:** 1.0 (MVP)
**Status:** Draft
**Owner:** Engineering
**Audience:** Engineering · DevOps · QA · Architecture Review
**Last Updated:** June 15, 2026
**Depends on:** JobCompass PRD v1.0

---

## 1. Purpose and Scope

This document translates the product requirements defined in the JobCompass PRD into concrete technical specifications. It defines system architecture, module-level technical design, API contracts, data models, infrastructure requirements, and non-functional requirements. It is the authoritative reference for engineering decisions during the MVP build.

---

## 2. System Architecture Overview

JobCompass is a multi-service web platform with a clear separation between the user-facing web application, the backend API, and the background job processing layer. The architecture is designed to:

- Run entirely on a single Docker Compose stack in development and early production
- Scale the scraping and AutoApplier layers independently without changing the application layer
- Remain fully self-hostable with no external SaaS dependencies beyond the LLM API and auth provider

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER                             │
│              Next.js 14 (App Router, TypeScript)                │
│         Dashboard · Job Tables · Resume Upload · Tracker        │
└─────────────────────────┬───────────────────────────────────────┘
                          │ HTTPS / REST + SSE
┌─────────────────────────▼───────────────────────────────────────┐
│                       API LAYER                                 │
│              FastAPI (Python 3.12, async)                       │
│    Auth · Profile · Jobs · Resume · Applications · Prep         │
└──────┬──────────────────┬──────────────────────────┬────────────┘
       │                  │                          │
┌──────▼──────┐   ┌───────▼────────┐   ┌────────────▼───────────┐
│  PostgreSQL  │   │  Redis          │   │  Cloudflare R2         │
│  (Primary   │   │  (Queue +       │   │  (File Storage:        │
│   DB)       │   │   Cache)        │   │   Resumes, Snapshots)  │
└─────────────┘   └───────┬────────┘   └────────────────────────┘
                          │ Task Queue
┌─────────────────────────▼───────────────────────────────────────┐
│                    WORKER LAYER (Celery)                        │
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────────────┐ │
│  │  Scraper     │  │  AutoApplier │  │  AI Pipeline          │ │
│  │  Workers     │  │  Workers     │  │  Workers              │ │
│  │  (Playwright)│  │  (Playwright)│  │  (Anthropic SDK)      │ │
│  └──────────────┘  └──────────────┘  └───────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────────────┐
│                   EXTERNAL SERVICES                             │
│  Anthropic API · Gov.uk Sponsor Register · Gmail IMAP           │
│  Company Career Sites (scraped)                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. Technology Stack — Detailed Justification

### 3.1 Backend: Python 3.12 + FastAPI

**Why FastAPI over Django/Flask:**
- Native `async/await` support — critical for concurrent scraping and LLM API calls
- Automatic OpenAPI spec generation (used by frontend for type-safe client generation)
- Pydantic v2 for request/response validation and serialisation
- Performance: ASGI-based, handles concurrent connections without threading overhead

**Key libraries:**
```
fastapi==0.115.x
uvicorn[standard]==0.30.x     # ASGI server
sqlalchemy[asyncio]==2.0.x    # ORM (async mode)
alembic==1.13.x               # DB migrations
celery[redis]==5.4.x          # Task queue
pydantic==2.7.x               # Data validation
pdfplumber==0.11.x            # PDF text extraction
python-docx==1.1.x            # DOCX parsing
playwright==1.44.x            # Browser automation
anthropic==0.28.x             # Claude API SDK
httpx==0.27.x                 # Async HTTP client
beautifulsoup4==4.12.x        # HTML parsing
python-jose[cryptography]     # JWT
passlib[bcrypt]               # Password hashing
boto3==1.34.x                 # S3/R2 file uploads
```

### 3.2 Frontend: Next.js 14 + TypeScript

**Why Next.js 14:**
- App Router with React Server Components for fast initial loads on job tables
- Built-in API routes for lightweight BFF (Backend for Frontend) pattern
- Static generation for marketing/landing pages, dynamic rendering for dashboards
- TypeScript throughout for type safety across the full stack

**Key libraries:**
```
next@14.x
typescript@5.x
tailwindcss@3.x
@shadcn/ui                    # Component library
recharts@2.x                  # Analytics charts
@tanstack/react-query@5.x     # Server state management
@tanstack/react-table@8.x     # Job listing tables
react-hook-form@7.x           # Form handling (profile, settings)
zod@3.x                       # Schema validation (shared with backend types)
axios@1.x                     # HTTP client
next-auth@5.x (Auth.js)       # OAuth + session management
```

### 3.3 Database: PostgreSQL 16 + pgvector

**Why PostgreSQL over alternatives:**
- Strong relational model for user → application → job relationships
- JSONB columns for flexible, semi-structured JD fields without a schema migration per new ATS
- `pgvector` extension for in-database vector similarity search (resume↔job matching) — eliminates a dedicated vector DB at MVP scale
- Full-text search via `tsvector` for keyword-based job filtering

**Extensions required:**
```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pgvector";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";
```

### 3.4 Task Queue: Redis + Celery

**Queue design:**
- Separate Celery queues per workload type to allow independent scaling:
  - `scraper` — low priority, high volume, rate-limited
  - `applier` — medium priority, sequential per user
  - `ai` — medium priority, LLM API calls
  - `email` — high priority, reply parsing
  - `default` — everything else

**Redis usage:**
- Celery broker and result backend
- Rate-limit counters (scraping per domain, AutoApplier per user)
- Session cache (for frequently accessed user profiles)
- Job deduplication Bloom filter keys

### 3.5 Browser Automation: Playwright + Stealth

**Configuration:**
```python
# Required for ATS and career site scraping
playwright-stealth==1.0.x    # Anti-detection patches
```

**Anti-detection measures:**
- Randomised user agents per session (maintained list of real Chrome UAs)
- Human-timing simulation: keystrokes 80–200ms, form navigation 300–800ms between fields
- Viewport randomisation (1280–1920px width, standard resolutions only)
- Headless: false in production workers (headful avoids most bot detection)
- Per-domain rate limiting: minimum 3 seconds between page loads on same domain

### 3.6 AI Layer: Anthropic Claude

**Model:** `claude-sonnet-4-6` for all tasks (balances speed, quality, and cost for MVP)

**Usage by module:**

| Module | Prompt type | Est. tokens in/out |
|---|---|---|
| Resume analysis | Long system prompt + resume text | 3000 in / 800 out |
| JD parsing | Structured extraction prompt | 1500 in / 400 out |
| Match scoring | Comparison prompt | 1000 in / 200 out |
| Cover letter gen | Profile + JD + tone instruction | 2000 in / 600 out |
| Screening Q answers | Profile + question + job context | 800 in / 200 out |
| Interview prep gen | JD + profile gaps + role seniority | 2500 in / 1200 out |

**Cost estimate (MVP):** ~$0.08–0.15 per full user session (resume analysis + 10 job matches + 3 applications)

---

## 4. Module Technical Specifications

### 4.1 Authentication & Session Management

**Flow:**
```
User → Next.js Auth.js → OAuth (Google/GitHub) OR email/password
     → API: POST /auth/session → JWT (access: 15min, refresh: 7d)
     → Stored in httpOnly cookie (not localStorage)
```

**JWT payload:**
```json
{
  "sub": "user_uuid",
  "email": "user@example.com",
  "role": "user",
  "iat": 1234567890,
  "exp": 1234568790
}
```

**Password policy:** Minimum 8 characters, bcrypt cost factor 12.

**Session storage:** Redis-backed server sessions for refresh token rotation. Invalidation on logout flushes Redis key immediately.

---

### 4.2 Resume Processing Pipeline

**Upload flow:**
```
Client → multipart/form-data → API: POST /resume/upload
→ Validate file (PDF/DOCX, max 5MB)
→ Store original in R2: resumes/{user_id}/{uuid}.{ext}
→ Enqueue: ai.resume_analyze task
→ Return: { task_id, status: "processing" }

Client polls: GET /resume/status/{task_id} (SSE stream)
→ Streams partial results as sections complete
```

**Text extraction:**
```python
# PDF
with pdfplumber.open(file_path) as pdf:
    text = "\n".join(page.extract_text() or "" for page in pdf.pages)

# DOCX
doc = docx.Document(file_path)
text = "\n".join(p.text for p in doc.paragraphs)
```

**Analysis pipeline steps:**
1. Extract raw text
2. Section detection (regex + LLM fallback): identify SUMMARY, EXPERIENCE, EDUCATION, SKILLS, PROJECTS
3. Structured extraction via Claude: parse each section into typed objects
4. ATS simulation: run text through a parser that mimics Workday/Greenhouse parsing rules
5. Scoring: compute 5 sub-scores (parsability, keywords, content, format, completeness)
6. Gap detection: compare extracted skills against user profile target role taxonomy
7. Recommendations: generate prioritised improvement list via Claude
8. Embedding: generate and store resume embedding vector (for match scoring)

**Scoring formula:**
```
ATS_score = (parsability × 0.25) + (keywords × 0.30) + (content × 0.20) 
          + (format × 0.15) + (completeness × 0.10)
```

---

### 4.3 Job Discovery & Scraping Engine

**Scrape target database:**
- Initial seed: UK sponsor register CSV (Home Office, updated weekly)
- Enrichment: curated list of 500+ UK tech employers (maintained in DB)
- Growth: auto-discovery via SERP scraping for "{role} careers uk" queries

**Scraper architecture:**

```
Scheduler (Celery Beat, every 6h)
  → Dispatch: scraper.crawl_company tasks (one per company)
    → Playwright: navigate to careers page
    → Detect: job listing page structure (pagination, infinite scroll, or static)
    → Extract: job cards → structured fields
    → Normalise: title, location, posted_at, salary, apply_url, contract_type
    → Dedup: check job_url hash against existing DB records
    → Upsert: insert new, update changed, mark stale inactive
    → Enqueue: ai.parse_jd task for new records only
```

**JD parsing (AI):**
```
Input: raw HTML/text of job description
Output: {
  required_skills: string[],
  preferred_skills: string[],
  seniority: "graduate" | "junior" | "mid" | "senior" | "lead",
  contract_type: "full-time" | "part-time" | "contract" | "graduate-scheme",
  salary_min: int | null,
  salary_max: int | null,
  remote_policy: "on-site" | "hybrid" | "remote",
  visa_sponsorship_mentioned: bool,
  description_clean: string  // stripped of boilerplate
}
```

**Sponsorship data pipeline:**
```
Weekly: fetch https://assets.publishing.service.gov.uk/...sponsors.csv
→ Parse CSV → upsert companies.is_registered_sponsor
→ Cross-reference with jobs table → update job.is_sponsorable

CoS count estimation (best effort):
→ Scrape employer immigration data from public transparency reports
→ Where unavailable: use company size + sector as proxy
→ Store confidence level: "confirmed" | "estimated" | "unknown"
```

**Salary estimation (where not stated in JD):**
```
Input: job title + seniority + company_size + location
Prompt: "Given a {seniority} {title} role at a {size} company in {location}, UK, 
         estimate the typical salary range in GBP. Return JSON: 
         {min: int, max: int, confidence: 'low'|'medium'|'high'}"
```

---

### 4.4 Job Search & Filtering API

**Endpoint:** `GET /jobs`

**Query parameters:**
```
board          : "graduate" | "sponsored" | "non-sponsored"
posted_within  : 24 | 48 | 72 | 168 | 336 (hours)
city           : string (UK city name)
radius_miles   : 10 | 25 | 50 | 100 | null (null = nationwide)
sponsor_tier   : "active" | "moderate" | "low" | "any"
salary_min     : int (GBP)
salary_max     : int (GBP)
remote_policy  : "on-site" | "hybrid" | "remote" | "any"
company_size   : "startup" | "sme" | "mid-market" | "enterprise" | "any"
sort           : "match_score" | "posted_at" | "salary_max"
page           : int (default: 1)
page_size      : int (default: 25, max: 100)
```

**Distance computation:**
- User selects a city → geocode to lat/lng (stored at query time)
- Jobs store company HQ lat/lng
- Filter via PostgreSQL: `earth_distance(ll_to_earth($lat, $lng), ll_to_earth(j.lat, j.lng)) <= $radius_metres`

**Match score injection:**
- If user has an active resume embedding, compute cosine similarity against each job's embedding at query time
- Materialised for performance: pre-computed match scores stored in `user_job_scores` table, refreshed when resume changes

---

### 4.5 AutoApplier — Technical Design

**State machine:**
```
QUEUED → STARTING → NAVIGATING → FILLING → REVIEWING → SUBMITTING → DONE
                                                    ↓
                                              CAPTCHA_PAUSE (→ user notified → resume)
                                                    ↓
                                              FAILED (→ logged, retry or skip)
```

**Form detection strategy:**
```python
ATS_SIGNATURES = {
    "greenhouse": ["boards.greenhouse.io", "gh_jid"],
    "lever":      ["jobs.lever.co", "lever-job"],
    "workday":    ["myworkdayjobs.com", "wd3.myworkdayjobs"],
    "icims":      ["icims.com"],
    "custom":     []  # fallback: generic field detection
}

def detect_ats(url: str, page_source: str) -> str:
    for ats, signals in ATS_SIGNATURES.items():
        if any(s in url or s in page_source for s in signals):
            return ats
    return "custom"
```

**Field mapping:**
```python
FIELD_MAP = {
    "first_name":   ["firstName", "first-name", "fname", "given-name"],
    "last_name":    ["lastName", "last-name", "lname", "family-name"],
    "email":        ["email", "emailAddress", "email-address"],
    "phone":        ["phone", "phoneNumber", "mobile"],
    "linkedin_url": ["linkedin", "linkedinUrl", "linkedin-url"],
    "cover_letter": ["coverLetter", "cover-letter", "coverLetterText"],
    "resume":       ["resume", "cv", "resumeFile"],
    # ... extended map per ATS
}
```

**Screening question answering:**
```python
async def answer_screening_question(question: str, user_profile: dict, job: dict) -> str:
    prompt = f"""
    You are filling out a job application for a {job['title']} role at {job['company']}.
    The applicant's profile: {json.dumps(user_profile)}
    Answer this screening question concisely and honestly:
    "{question}"
    Return only the answer text, no preamble.
    """
    response = await anthropic.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=300,
        messages=[{"role": "user", "content": prompt}]
    )
    return response.content[0].text
```

**Document snapshot on submission:**
```python
# Store exactly what was sent — critical for audit and follow-up
snapshot = ApplicationSnapshot(
    application_id=application.id,
    resume_version_id=resume.version_id,
    resume_text=resume.extracted_text,
    cover_letter_text=cover_letter,
    screening_answers=answers_dict,
    submitted_at=datetime.utcnow(),
    ats_detected=ats_type,
    submission_screenshot_url=screenshot_r2_url
)
```

---

### 4.6 Match & Scoring Engine

**Embedding model:** `text-embedding-3-small` (OpenAI) or Anthropic Embeddings
**Vector dimensions:** 1536 (OpenAI) / stored in pgvector column

**Match score computation:**
```python
def compute_match_score(resume_embedding: list[float], job_embedding: list[float],
                        resume_skills: list[str], job_required_skills: list[str],
                        resume_seniority: str, job_seniority: str) -> dict:
    
    # Semantic similarity (60% weight)
    semantic_score = cosine_similarity(resume_embedding, job_embedding) * 100
    
    # Keyword overlap (30% weight)
    resume_skills_set = set(s.lower() for s in resume_skills)
    job_skills_set = set(s.lower() for s in job_required_skills)
    keyword_score = len(resume_skills_set & job_skills_set) / max(len(job_skills_set), 1) * 100
    
    # Seniority fit (10% weight)
    SENIORITY_LEVELS = ["graduate", "junior", "mid", "senior", "lead"]
    res_level = SENIORITY_LEVELS.index(resume_seniority)
    job_level = SENIORITY_LEVELS.index(job_seniority)
    seniority_score = max(0, 100 - abs(res_level - job_level) * 25)
    
    final_score = (semantic_score * 0.6) + (keyword_score * 0.3) + (seniority_score * 0.1)
    
    return {
        "match_score": round(final_score, 1),
        "semantic_score": round(semantic_score, 1),
        "keyword_score": round(keyword_score, 1),
        "seniority_score": round(seniority_score, 1),
        "matched_skills": list(resume_skills_set & job_skills_set),
        "missing_skills": list(job_skills_set - resume_skills_set)
    }
```

---

### 4.7 Email Parsing (Application Tracker)

**Integration:** Gmail API (OAuth2) + IMAP fallback for non-Gmail users

**Status detection rules:**
```python
EMAIL_STATUS_PATTERNS = {
    "screening":  ["phone screen", "call scheduled", "initial interview", "recruiter call"],
    "interview":  ["interview invitation", "we'd like to invite", "interview slot"],
    "offer":      ["pleased to offer", "offer letter", "congratulations"],
    "rejected":   ["unfortunately", "not moving forward", "other candidates", "position filled"],
    "viewed":     []  # inferred from open tracking pixel if present
}

def classify_email(subject: str, body: str) -> str | None:
    text = (subject + " " + body).lower()
    for status, patterns in EMAIL_STATUS_PATTERNS.items():
        if any(p in text for p in patterns):
            return status
    return None  # no status update detected
```

---

## 5. Database Schema

### 5.1 Core Tables

```sql
-- Users
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email           VARCHAR(255) UNIQUE NOT NULL,
    password_hash   VARCHAR(255),
    oauth_provider  VARCHAR(50),
    oauth_sub       VARCHAR(255),
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);

-- User Profiles
CREATE TABLE user_profiles (
    id                  UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id             UUID REFERENCES users(id) ON DELETE CASCADE,
    full_name           VARCHAR(255),
    current_title       VARCHAR(255),
    years_experience    SMALLINT,
    target_title        VARCHAR(255),
    target_salary_min   INTEGER,
    target_salary_max   INTEGER,
    preferred_cities    VARCHAR(255)[],
    work_eligibility    VARCHAR(50),  -- 'uk_citizen' | 'settled' | 'visa_required'
    visa_type           VARCHAR(50),
    skills              VARCHAR(100)[],
    seniority_level     VARCHAR(20),
    theme               VARCHAR(20) DEFAULT 'system',
    accent_color        VARCHAR(7),
    updated_at          TIMESTAMPTZ DEFAULT NOW()
);

-- Resumes
CREATE TABLE resumes (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id         UUID REFERENCES users(id) ON DELETE CASCADE,
    version_number  SMALLINT NOT NULL DEFAULT 1,
    file_url        VARCHAR(500),
    file_name       VARCHAR(255),
    file_type       VARCHAR(10),
    raw_text        TEXT,
    parsed_data     JSONB,
    embedding       VECTOR(1536),
    ats_score       SMALLINT,
    score_breakdown JSONB,
    gaps            JSONB,
    recommendations JSONB,
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Companies
CREATE TABLE companies (
    id                      UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name                    VARCHAR(255) NOT NULL,
    domain                  VARCHAR(255) UNIQUE,
    careers_url             VARCHAR(500),
    size_band               VARCHAR(20),
    sector                  VARCHAR(100),
    hq_city                 VARCHAR(100),
    lat                     DECIMAL(9,6),
    lng                     DECIMAL(9,6),
    is_registered_sponsor   BOOLEAN DEFAULT FALSE,
    sponsor_confirmed_at    TIMESTAMPTZ,
    cos_6m_count            SMALLINT,
    cos_12m_count           SMALLINT,
    cos_count_confidence    VARCHAR(20) DEFAULT 'unknown',
    last_scraped_at         TIMESTAMPTZ,
    scrape_enabled          BOOLEAN DEFAULT TRUE,
    created_at              TIMESTAMPTZ DEFAULT NOW(),
    updated_at              TIMESTAMPTZ DEFAULT NOW()
);

-- Jobs
CREATE TABLE jobs (
    id                  UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    company_id          UUID REFERENCES companies(id),
    external_id         VARCHAR(255),
    title               VARCHAR(255) NOT NULL,
    description_raw     TEXT,
    description_clean   TEXT,
    parsed_data         JSONB,
    embedding           VECTOR(1536),
    location            VARCHAR(255),
    lat                 DECIMAL(9,6),
    lng                 DECIMAL(9,6),
    remote_policy       VARCHAR(20),
    contract_type       VARCHAR(30),
    seniority           VARCHAR(20),
    board               VARCHAR(20),       -- 'graduate' | 'sponsored' | 'non-sponsored'
    is_sponsorable      BOOLEAN,
    salary_min          INTEGER,
    salary_max          INTEGER,
    salary_currency     CHAR(3) DEFAULT 'GBP',
    salary_confidence   VARCHAR(10),
    apply_url           VARCHAR(500) NOT NULL,
    source_domain       VARCHAR(255),
    posted_at           TIMESTAMPTZ,
    scraped_at          TIMESTAMPTZ DEFAULT NOW(),
    is_active           BOOLEAN DEFAULT TRUE,
    url_hash            VARCHAR(64) UNIQUE,
    UNIQUE(company_id, external_id)
);

-- Applications
CREATE TABLE applications (
    id                  UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id             UUID REFERENCES users(id) ON DELETE CASCADE,
    job_id              UUID REFERENCES jobs(id),
    resume_id           UUID REFERENCES resumes(id),
    status              VARCHAR(30) DEFAULT 'saved',
    applied_via         VARCHAR(20),       -- 'autoapplier' | 'manual'
    applied_at          TIMESTAMPTZ,
    match_score         DECIMAL(5,2),
    notes               TEXT,
    created_at          TIMESTAMPTZ DEFAULT NOW(),
    updated_at          TIMESTAMPTZ DEFAULT NOW()
);

-- Application Snapshots (audit)
CREATE TABLE application_snapshots (
    id                      UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    application_id          UUID REFERENCES applications(id) ON DELETE CASCADE,
    resume_text             TEXT,
    cover_letter_text       TEXT,
    screening_answers       JSONB,
    ats_detected            VARCHAR(30),
    submission_screenshot   VARCHAR(500),
    submitted_at            TIMESTAMPTZ DEFAULT NOW()
);

-- Application Status History
CREATE TABLE application_status_history (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    application_id  UUID REFERENCES applications(id) ON DELETE CASCADE,
    from_status     VARCHAR(30),
    to_status       VARCHAR(30) NOT NULL,
    changed_at      TIMESTAMPTZ DEFAULT NOW(),
    changed_by      VARCHAR(20),  -- 'user' | 'email_parser' | 'autoapplier'
    note            TEXT
);

-- User Job Match Scores (materialised)
CREATE TABLE user_job_scores (
    user_id         UUID REFERENCES users(id) ON DELETE CASCADE,
    job_id          UUID REFERENCES jobs(id) ON DELETE CASCADE,
    match_score     DECIMAL(5,2),
    score_breakdown JSONB,
    computed_at     TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (user_id, job_id)
);

-- Interview Prep
CREATE TABLE interview_preps (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id         UUID REFERENCES users(id) ON DELETE CASCADE,
    application_id  UUID REFERENCES applications(id),
    job_id          UUID REFERENCES jobs(id),
    prep_content    JSONB,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Scrape Queue
CREATE TABLE scrape_queue (
    id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    company_id      UUID REFERENCES companies(id),
    status          VARCHAR(20) DEFAULT 'pending',
    scheduled_for   TIMESTAMPTZ DEFAULT NOW(),
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    jobs_found      INTEGER,
    error           TEXT
);
```

### 5.2 Key Indexes

```sql
-- Job discovery queries
CREATE INDEX idx_jobs_board_active       ON jobs(board, is_active);
CREATE INDEX idx_jobs_posted_at          ON jobs(posted_at DESC);
CREATE INDEX idx_jobs_company            ON jobs(company_id);
CREATE INDEX idx_jobs_sponsorable        ON jobs(is_sponsorable) WHERE is_active = TRUE;
CREATE INDEX idx_jobs_location           ON jobs USING GIST(ll_to_earth(lat, lng));

-- Full-text search
CREATE INDEX idx_jobs_fts ON jobs USING GIN(to_tsvector('english', title || ' ' || COALESCE(description_clean, '')));

-- Vector search
CREATE INDEX idx_jobs_embedding          ON jobs USING ivfflat(embedding vector_cosine_ops);
CREATE INDEX idx_resumes_embedding       ON resumes USING ivfflat(embedding vector_cosine_ops);

-- Application lookups
CREATE INDEX idx_applications_user       ON applications(user_id);
CREATE INDEX idx_applications_status     ON applications(user_id, status);
CREATE INDEX idx_match_scores_user       ON user_job_scores(user_id, match_score DESC);
```

---

## 6. API Contract Summary

### 6.1 Base URL and Versioning
```
Development:  http://localhost:8000/api/v1
Production:   https://api.jobcompass.co.uk/api/v1
```

### 6.2 Core Endpoints

```
AUTH
POST   /auth/register          Register with email/password
POST   /auth/login             Login, returns JWT
POST   /auth/logout            Invalidate refresh token
POST   /auth/refresh           Refresh access token
GET    /auth/oauth/{provider}  OAuth redirect (google, github)

PROFILE
GET    /profile                Get current user profile
PUT    /profile                Update profile
DELETE /profile                Delete account + all data

RESUME
POST   /resume/upload          Upload resume file → async analysis
GET    /resume                 Get current active resume + scores
GET    /resume/history         List all resume versions
GET    /resume/status/{task}   Poll analysis status (SSE stream)
DELETE /resume/{id}            Delete a resume version

JOBS
GET    /jobs                   Query jobs (all filters as query params)
GET    /jobs/{id}              Get single job detail
GET    /jobs/{id}/match        Get match score for a specific job
POST   /jobs/{id}/save         Save job to list
DELETE /jobs/{id}/save         Unsave job

APPLICATIONS
POST   /applications           Create application (manual log)
GET    /applications           List all applications with status
GET    /applications/{id}      Get single application + history + snapshot
PUT    /applications/{id}      Update status or notes
DELETE /applications/{id}      Remove from tracker

AUTOAPPLIER
POST   /autoapplier/queue      Queue job(s) for auto-apply
GET    /autoapplier/status     Get queue status + active tasks
PUT    /autoapplier/pause      Pause the queue
PUT    /autoapplier/resume     Resume the queue
GET    /autoapplier/config     Get user's AutoApplier config
PUT    /autoapplier/config     Update config (caps, threshold, schedule)

INTERVIEW PREP
POST   /prep/generate          Generate prep guide for application/job
GET    /prep/{id}              Get a prep guide
GET    /prep                   List all prep guides

ANALYTICS
GET    /analytics/summary      Overview stats
GET    /analytics/funnel       Application funnel by status
GET    /analytics/boards       Response rate by job board
GET    /analytics/scores       Match score vs. response rate correlation
```

---

## 7. Non-Functional Requirements

### 7.1 Performance

| Metric | Target |
|---|---|
| Job listing page load (cold) | < 1.5s |
| Job listing page load (cached) | < 300ms |
| Resume analysis (end-to-end) | < 30s (streamed progressively) |
| Match score computation (per job) | < 50ms (pre-computed) |
| AutoApplier form submission | < 3 minutes per application |
| API p95 response time | < 500ms |
| Job data freshness | ≥ 80% of listings updated within 12h of source |

### 7.2 Reliability

| Metric | Target |
|---|---|
| API uptime | 99.5% (MVP) |
| AutoApplier success rate | > 90% of non-CAPTCHA submissions |
| Scraper coverage | > 85% of target company list actively scraped |
| Email parser accuracy | > 80% correct status classification |

### 7.3 Security

- All file uploads scanned for MIME type (not just extension) — reject non-PDF/DOCX
- Resume files stored in private R2 bucket, accessed via signed URLs (1h TTL)
- Rate limiting: 100 req/min per authenticated user, 10 req/min per IP for auth endpoints
- OWASP Top 10 compliance: parameterised queries throughout, no raw SQL with user input
- Secrets in environment variables only — never committed to repo
- GDPR-compliant: user can export all data (`GET /profile/export`) and delete all data (`DELETE /profile`)

### 7.4 Scalability

- Scraper workers are stateless — scale horizontally behind Celery
- AutoApplier workers isolated per user — one active Playwright session per user
- Database connection pooling via PgBouncer (MVP) or SQLAlchemy pool
- CDN (Cloudflare) in front of frontend — static assets cached at edge

---

## 8. Infrastructure & DevOps

### 8.1 Development Stack (Docker Compose)

```yaml
# infra/docker-compose.yml
services:
  db:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_DB: jobcompass
      POSTGRES_USER: jobcompass
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    ports: ["5432:5432"]
    volumes: ["postgres_data:/var/lib/postgresql/data"]

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

  api:
    build: ./backend
    command: uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
    ports: ["8000:8000"]
    environment:
      DATABASE_URL: postgresql+asyncpg://jobcompass:${DB_PASSWORD}@db/jobcompass
      REDIS_URL: redis://redis:6379
      ANTHROPIC_API_KEY: ${ANTHROPIC_API_KEY}
      R2_BUCKET: ${R2_BUCKET}
      R2_ACCESS_KEY: ${R2_ACCESS_KEY}
      R2_SECRET_KEY: ${R2_SECRET_KEY}
    depends_on: [db, redis]
    volumes: ["./backend:/app"]

  worker-scraper:
    build: ./backend
    command: celery -A app.tasks worker -Q scraper -c 4 --loglevel=info
    depends_on: [db, redis]

  worker-ai:
    build: ./backend
    command: celery -A app.tasks worker -Q ai -c 2 --loglevel=info
    depends_on: [db, redis]

  worker-applier:
    build: ./backend
    command: celery -A app.tasks worker -Q applier -c 1 --loglevel=info
    depends_on: [db, redis]

  beat:
    build: ./backend
    command: celery -A app.tasks beat --loglevel=info
    depends_on: [redis]

  frontend:
    build: ./frontend
    command: npm run dev
    ports: ["3000:3000"]
    volumes: ["./frontend:/app", "/app/node_modules"]
    environment:
      NEXT_PUBLIC_API_URL: http://localhost:8000/api/v1

volumes:
  postgres_data:
```

### 8.2 Production Deployment (MVP)

**Platform:** Railway or Render (lowest ops overhead for solo/small team)

**Services to deploy:**
- `api` — FastAPI (1x, autoscale on CPU)
- `worker-scraper` — Celery scraper (1–3x)
- `worker-ai` — Celery AI (1x)
- `worker-applier` — Celery applier (1x per 10 active users)
- `beat` — Celery beat scheduler (1x, never scale)
- PostgreSQL — managed (Railway Postgres or Supabase)
- Redis — managed (Railway Redis or Upstash)
- Frontend — Vercel (Next.js, zero config)

### 8.3 Environment Variables

```env
# Database
DATABASE_URL=postgresql+asyncpg://...
REDIS_URL=redis://...

# AI
ANTHROPIC_API_KEY=sk-ant-...
OPENAI_API_KEY=sk-...  # for embeddings (or use Anthropic)

# File Storage (Cloudflare R2)
R2_ENDPOINT_URL=https://<account>.r2.cloudflarestorage.com
R2_ACCESS_KEY=...
R2_SECRET_KEY=...
R2_BUCKET=jobcompass-files

# Auth
NEXTAUTH_SECRET=...
NEXTAUTH_URL=https://jobcompass.co.uk
GOOGLE_CLIENT_ID=...
GOOGLE_CLIENT_SECRET=...
GITHUB_CLIENT_ID=...
GITHUB_CLIENT_SECRET=...

# Email
GMAIL_CLIENT_ID=...
GMAIL_CLIENT_SECRET=...

# Feature flags
MAX_APPLICATIONS_PER_DAY=10
MATCH_SCORE_THRESHOLD=65
SCRAPE_INTERVAL_HOURS=6
```

---

## 9. Error Handling Standards

### 9.1 API Error Response Format

All errors return a consistent shape:

```json
{
  "error": {
    "code": "RESUME_TOO_LARGE",
    "message": "Resume file must be under 5MB",
    "detail": null,
    "request_id": "req_abc123"
  }
}
```

### 9.2 Error Code Registry

```
AUTH_001  Invalid credentials
AUTH_002  Token expired
AUTH_003  Insufficient permissions

RESUME_001  Unsupported file type (must be PDF or DOCX)
RESUME_002  File size exceeds 5MB limit
RESUME_003  Analysis failed — text extraction returned empty
RESUME_004  No active resume found

JOBS_001  Invalid filter parameter
JOBS_002  Job no longer active

APPLIER_001  Daily application cap reached
APPLIER_002  Match score below threshold
APPLIER_003  CAPTCHA detected — manual intervention required
APPLIER_004  ATS form structure not recognised
APPLIER_005  Submission failed after 3 retries

SCRAPER_001  Career page not accessible (403/404)
SCRAPER_002  Rate limited by target domain
```

---

## 10. Testing Strategy

### 10.1 Coverage targets

| Layer | Type | Target coverage |
|---|---|---|
| Backend API | Unit (pytest) | 80% |
| Backend API | Integration (TestClient) | Core endpoints 100% |
| Scraper | Unit (mock pages) | 70% |
| AutoApplier | Integration (Playwright test sites) | 3 ATS types |
| Match engine | Unit (known pairs) | 90% |
| Frontend | Component (Vitest) | 60% |
| E2E | Playwright (critical paths) | 5 flows |

### 10.2 Critical E2E test flows

1. Register → Upload resume → View ATS score
2. Upload resume → Search jobs → Filter by sponsored → View match scores
3. Select job → AutoApplier → Verify application created in tracker
4. Tracker → Receive simulated recruiter email → Status auto-updates
5. Application → Generate interview prep guide

---

## 11. Open Technical Decisions

| Decision | Options | Recommendation | By when |
|---|---|---|---|
| Embedding provider | Anthropic vs OpenAI | OpenAI text-embedding-3-small (lower cost, same quality for this use case) | Before W2 |
| Vector search scale | pgvector vs Qdrant | pgvector for MVP; migrate if >500k jobs | W4 review |
| Stealth scraping | DIY vs Browserless.io | DIY for MVP; Browserless if block rate >15% | W1 |
| Auth provider | Auth.js vs Supabase Auth | Auth.js (more control, no vendor lock) | W1 |
| Gov.uk data format | CSV vs HTML scrape | CSV (stable, officially published format) | W1 |
| CoS count data | Transparency reports vs proxy model | Proxy model for MVP (LLM estimate from company size/sector) | W2 |
