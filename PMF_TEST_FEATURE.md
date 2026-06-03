# Agentic PMF Test — Feature Description

_Written: 2026-06-03. Repo: `agentic-pmf-test`. Status: active development._

---

## 1. What It Is

The **Agentic PMF Test** is a two-phase tool that answers one question for consumer subscription app teams:

> *"Does this product have what it takes to become a profitable subscription brand — and can we prove it with data?"*

It starts with a free AI-powered pre-campaign score (Phase 1) and gates into a live campaign test (Phase 2). Phase 1 is being built now. Phase 2 is a future capability.

---

## 2. Why We're Building It

FloKit's positioning is around the full growth infrastructure for consumer subscription apps. Most teams arrive at FloKit after they've already wasted money on campaigns that didn't work, or built paywalls for products that had positioning problems no campaign could fix.

The PMF Test is the entry point. It captures teams **before** they spend, gives them a signal worth trusting, and creates a natural path into FloKit's deeper tooling.

---

## 3. Target Users

**Primary:**
- Consumer subscription app founders (pre-launch or early stage)
- Growth leads at live subscription apps that have stalled
- App studio operators validating a new product idea
- Indie app builders who need research-grade insight without a research team

**Best-fit app categories:**
Health & fitness, lifestyle, productivity, utilities, education, photo/video, AI apps, wellness, learning, dating/social, entertainment, digital goods.

**Less ideal:**
Games, pure e-commerce, physical goods, local services, heavily regulated verticals.

---

## 4. Account Setup

Minimum required: **email + company name**. That's the full onboarding.

Authentication:
- Magic link via **Resend** (primary — no password required)
- Google OAuth (secondary)

No App Store Connect, no Google Play Console, no ad account credentials, no card — ever in the free flow.

---

## 5. Phase 1 — Pre-Campaign Score (being built now)

### 5.1 Input

User provides at least one of:
- Product website URL
- App Store URL
- Google Play URL
- Written product description (min 20 chars)

### 5.2 Data Collection

1. All submitted URLs are scraped via **Firecrawl** (returns clean markdown, handles JS-rendered pages)
2. If a product URL is found, the `/pricing` sub-path is also scraped automatically
3. Coverage level is assessed (`high` / `medium` / `low`) based on total character yield
4. Low coverage triggers a confidence warning in the output — no fake precision

All data is public-only. No credentials are requested.

### 5.3 Scoring — Five Pillars (0–20 each → 0–100 total)

Each pillar is evaluated by **GPT-4o** against a structured rubric. Scores are clamped to 0–20.

#### Pillar 1 — Value Proposition (0–20)
Evaluates the core promise extracted from website/listing copy.

| Sub-criterion | Max | What earns full marks |
|---|---|---|
| Specificity of outcome | 7 | Concrete, measurable result stated (e.g. "lose 5kg in 8 weeks") |
| Credibility signals | 7 | Testimonials, data, press mentions, awards present |
| Differentiation from category language | 6 | Copy doesn't sound identical to the top 5 competitors |

#### Pillar 2 — Market Positioning (0–20)
Classifies the app category and evaluates white-space.

| Sub-criterion | Max | What earns full marks |
|---|---|---|
| Niche clarity | 8 | Specific sub-niche with clear ICP defined |
| Identifiable white space | 7 | A gap vs. the category that this app could plausibly own |
| Category growth signal | 5 | Emerging or growing category (not saturated/declining) |

#### Pillar 3 — Pricing Confidence (0–20)
Evaluates pricing page structure against subscription app benchmarks.

| Sub-criterion | Max | What earns full marks |
|---|---|---|
| Plan structure clarity | 6 | 1–3 clearly presented plans |
| Trial offer | 5 | Free trial or money-back guarantee present |
| Annual plan discount | 5 | Annual option with 30–50% discount |
| Price/category alignment | 4 | Within ~2× of category median |

If no pricing page is found, score is conservative and confidence is flagged.

#### Pillar 4 — Audience Clarity (0–20)
Evaluates how specifically the target user is described on public pages.

| Sub-criterion | Max | What earns full marks |
|---|---|---|
| Named target user | 7 | Specific persona described (not "anyone who wants to be healthier") |
| Explicit pain point | 7 | Specific and emotionally resonant pain stated |
| Concrete use-case scenario | 6 | A specific situation described (not abstract benefit) |

#### Pillar 5 — Competitive Gap (0–20)
GPT-4o uses its category knowledge to assess differentiation.

| Sub-criterion | Max | What earns full marks |
|---|---|---|
| Uniqueness of core claim | 8 | No top competitor makes this exact claim |
| Category saturation | 6 | Category still has room for a new entrant |
| Defensibility | 6 | Positioning would be hard to copy quickly |

### 5.4 Readiness Tiers

| Score | Label | What it means |
|---|---|---|
| 65–100 | Strong Phase 2 candidate | GPT generates a recommended test angle |
| 40–64 | Fixable | Top 3 gaps listed with recommended fixes |
| 0–39 | Not ready | Product-market fit work needed first |

### 5.5 Report Output

- **FloKit.AI Score** (0–100) with all five pillar breakdowns
- Per-pillar: score, one-line reason, signals present/missing
- **Verdict** (1–2 sentences, GPT-generated, specific to this product)
- **Recommended angle** for Phase 2 (if score ≥ 65)
- **Top 3 risks** and **top 3 opportunities**
- Confidence level indicator (high / medium / low based on scrape coverage)
- PDF export (generated async, stored in S3)

---

## 6. Phase 2 — Live Campaign Test (future, not being built now)

Phase 2 is presented as a locked/coming-soon section on the marketing page. No backend code is written for Phase 2 in this iteration.

When built, Phase 2 will:
1. Generate a DTC-ready brand and store from the product inputs
2. Generate 10–30 ad creatives across angles and hooks
3. Launch campaigns on Meta and TikTok with real budget
4. Collect CPM, CTR, hold rate, and engagement signals
5. Return a final FloKit.AI Score incorporating real market data

Phase 2 is gated: only users whose Phase 1 score is ≥ 65 will be offered Phase 2.

---

## 7. Architecture

```
/website         — React marketing site (PMF Test landing section + signup flow)
agentic-pmf-test — Backend API + workers only

agentic-pmf-test/
  src/
    server/         Hono HTTP API (auth, score routes)
    workers/        BullMQ pipeline (scrape → score → store)
    shared/         Types, logger, db, jwt
  k8s/
  .github/workflows/ci.yml
  Dockerfile
```

**Stack:** TypeScript · Hono · Node 22 · OpenAI GPT-4o · MongoDB · BullMQ + Redis · Firecrawl · Resend · S3

---

## 8. Auth Flow

1. User enters email + company name on the marketing site
2. POST `/api/auth/magic-link` → Resend sends a signed 32-char token link (15 min TTL)
3. User clicks link → GET `/api/auth/verify?token=...` → returns JWT (30-day expiry)
4. JWT stored in `localStorage`, sent as `Authorization: Bearer <token>` on all API calls
5. Google OAuth available as an alternative (same result: JWT issued)

---

## 9. Data & Privacy

- Only public data is used unless the user explicitly connects private tools
- No App Store Connect, no Play Console, no ad account credentials in this flow
- PII (email) is never logged — only hashed IDs and domain names in logs
- Magic tokens are single-use and expire after 15 minutes

---

## 10. CI/CD

Follows `CICD_GUIDELINES.md` exactly:
- PR pipeline: typecheck → test → coverage → audit → docker build
- Merge to `dev` → auto-deploy to sandbox
- Merge to `master` (via reviewed PR) → auto-deploy to production
- GitHub Actions, ECR, EKS
- Coralogix for logs, Grafana for metrics

---

## 11. Success Metrics (Phase 1)

| Metric | Target |
|---|---|
| Signup → score completion rate | > 60% |
| Report satisfaction (thumbs up) | > 70% |
| Score → Phase 2 waitlist click | > 25% for scores ≥ 65 |
| Company-domain email capture | > 40% of signups |
