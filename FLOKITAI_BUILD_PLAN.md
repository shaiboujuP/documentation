# FloKit.AI — Architecture & Logic Design & Phased Build Plan

**Internal Design Document**  
**Date:** May 30, 2026

---

## Table of Contents

1. [Platform Mental Model](#1-platform-mental-model)
2. [Core Data Model](#2-core-data-model)
3. [Feature-by-Feature Design](#3-feature-by-feature-design)
4. [System Integration Architecture](#4-system-integration-architecture)
5. [Agent Orchestration Layer](#5-agent-orchestration-layer)
6. [Compliance & Trust Layer](#6-compliance--trust-layer)
7. [Key Design Tensions](#7-key-design-tensions)
8. [Phased Build Plan](#8-phased-build-plan)
   - [Phase 0 — Foundation](#phase-0--foundation-34-weeks)
   - [Phase 1 — Core Services](#phase-1--core-services-56-weeks)
   - [Phase 2 — Integration & Intelligence](#phase-2--integration--intelligence-56-weeks)
   - [Phase 3 — AI Agent Layer](#phase-3--ai-agent-layer-57-weeks)
   - [Phase 4 — Advanced Commerce & Growth](#phase-4--advanced-commerce--growth-45-weeks)
9. [Critical Path](#9-critical-path)
10. [Phase Summary](#10-phase-summary)
11. [Staffing Allocation](#11-staffing-allocation)

---

## 1. Platform Mental Model

FloKit.AI is an **agentic DTC (Direct-to-Consumer) infrastructure platform** for consumer subscription apps. The central insight: every traditionally manual step in the funnel (ad creative, paywall, offer selection, lifecycle email) can be run by an autonomous agent that observes real signals and acts continuously.

**Three rails:**
- **Acquisition rail** — get users from channels into the app
- **Monetization rail** — convert users to paying subscribers
- **Lifecycle rail** — retain, re-engage, and upsell

All three rails share one data bus and are orchestrated by agents, not human operators on dashboards.

---

## 2. Core Data Model

### Entities

| Entity | Key Fields |
|---|---|
| **App** | Belongs to Team; has many Campaigns, Paywalls, Offers, Events, Users |
| **User** | `device_id`, `email`, `attributed_channel`, `state` (anonymous/trial/subscribed/churned/winback), `segment_ids` |
| **Offer** | `type` (trial/annual/monthly/winback/upsell), `price_point`, `currency`, `creative_ref`, `conditions` |
| **Campaign** | `channel`, `creative_set`, `target_audience_spec`, `budget`, `bidding_model` |
| **Paywall** | `layout_config`, `active_offers`, `experiment`, `personalization_rules` |
| **Subscription** | `user → product → price`, `status`, `MoR_provider`, `lifecycle_events` |

---

## 3. Feature-by-Feature Design

### 3.1 AI Offer Agent

**What it does:** Autonomously selects which offer (price, creative, timing, channel) to show a given user in real time.

**Logic flow:**
1. Trigger fires (user hits paywall, enters chatbot, or email event fires)
2. Signal collection: user segment, session context, attribution source, historical offer performance, current A/B test arms
3. Feature vector → Offer Scoring Model: input is user segment + context + offer catalog; output is ranked list of `(offer, creative, channel)` tuples with confidence scores
4. Dispatch decision: high confidence → auto-dispatch; low confidence → escalate to experiment arm; log decision + outcome for retraining
5. Outcome tracking: conversion event → reward signal; skip/dismiss → negative signal; batch retraining loop (daily or on-threshold)

**Key design decisions:**
- Agent operates on a **contextual bandit algorithm** — simpler to reason about, faster to converge in subscription contexts
- Guardrails: min/max price boundaries, legal compliance per region, A/B experiment allocation caps
- Humans set the offer catalog and guardrails; the agent picks from that space

---

### 3.2 Subscription Offer Builder

**What it does:** Auto-generates trial, annual, and win-back pricing structures, ready for experimentation.

**Logic:**
1. **Input:** app category, competitor pricing, historical conversion rates, MoR provider capabilities
2. **Generation:** templates `{trial_days, monthly_price, annual_price, discount_pct}`, heuristics by vertical, LLM drafts offer copy, outputs 3–5 offer candidates per type
3. **Experiment scaffolding:** each candidate auto-assigned an experiment arm, traffic allocation recommended, statistical significance threshold configured
4. **MoR sync:** pushes approved offers to Stripe/RevenueCat/Paddle via API

---

### 3.3 Web-to-App Flow

**What it does:** Rebuilds the landing page → App Store → first session handoff as a first-class instrumented flow.

**Flow:**
1. Paid ad / organic search
2. Personalized landing page (variant selected by channel, creative, user segment)
3. Checkout path split:
   - **Path A (web checkout):** user pays on web, deeplink carries auth token into app
   - **Path B (App Store):** redirect with SKAdNetwork attribution + deferred deeplink
4. App install + first session: deferred deeplink resolves, attribution joined (`web_session_id → install → first_open`), events fired
5. Leak detection: drop-off measured at each step; alert if web→App Store conversion < threshold; agent can auto-rotate landing page variant

**Attribution model:** first-touch (channel credit), last-touch (landing page variant credit), multi-touch (planned)

---

### 3.4 Acquisition & Creative System

**Channels supported:** AppLovin, Meta/Instagram, Facebook, AI surfaces, ASO, organic web.

**Creative generation pipeline:**
1. **Input sources:** App Store screenshots/metadata/description, app catalog ingestion, brand kit, top-performing creative signals
2. **Generation:** LLM generates copy variants (hooks, CTAs, body); image/video model generates visual assets; composition overlays copy on visuals per channel spec; outputs N variants per format (static, video, carousel)
3. **Campaign builder:** selects audience spec, assigns creatives to ad groups, sets bidding model, pushes to channel API
4. **Iteration loop:** performance data pulled daily, underperformers paused automatically, new variants generated to replace them, agent proposes budget reallocation

**Revenue-share UA financing model:** developer gets upfront UA spend from FloKit; FloKit recoups from a % of revenue generated by those installs; risk model uses cohort LTV prediction at D3/D7/D30 to approve financing.

---

### 3.5 AI Commerce & Agent Commerce Routes

**What it does:** Enables purchases to happen anywhere — in-app paywalls, chatbots, LLM interfaces, landing pages, email.

**Offer surface registry:**
- In-app paywall (native SDK)
- Web landing page (JS snippet)
- Chatbot (webhook / MCP integration)
- Email (rendered offer card + one-click link)
- LLM surface (MCP tool that returns offer payload)

**One-tap checkout:** user identity resolved → payment method (saved card, Apple Pay, Google Pay) → MoR processes payment → entitlement provisioned → deeplink into app.

**MCP integration for LLM surfaces:** FloKit exposes a `get_offer(user_context)` MCP tool; LLM calls tool during conversation, receives offer JSON, renders offer inline; user clicks → one-tap checkout webhook.

---

### 3.6 Monetization Console

**What it does:** Central operator view. Shows what agents are doing, lets humans inspect and override.

**Components:**
- **Agent activity feed:** recent decisions, alerts, recommendations
- **Live signal dashboard:** active sessions → paywalls shown → conversions (real-time); revenue by channel/variant/segment; funnel waterfall
- **Offer management:** catalog editor, experiment status, guardrail editor
- **Creative library:** all generated assets with performance overlay
- **Human override:** pause any agent decision category, force specific offer to 100% traffic, full audit log

---

### 3.7 Lifecycle Engine

**What it does:** Post-conversion retention via email, in-app messaging, and re-engagement.

**User state machine:**

```
trial_started
  → Day 3: engagement check (sessions < 2 → "activation" email)
  → Day 6: conversion nudge (paywall soft-push)
  → Day 7: trial end → convert or expire

subscribed
  → Renewal approaching (3 days) → confirmation email
  → Renewal failed → dunning sequence (email D1, D3, D7)
  → Low usage detected → re-engagement push

churned
  → Day 7 post-churn → win-back offer (discount)
  → Day 30 → final win-back (steeper discount or free week)
  → Day 60 → archive (no further contact)
```

**Email system:** LLM personalizes subject + body per segment, send-time optimization, CAN-SPAM and GDPR compliance.

---

## 4. System Integration Architecture

```
External inputs                FloKit.AI core                 External outputs
─────────────────             ────────────────               ─────────────────
Ad channels          ──→      Acquisition Engine    ──→      Channel APIs
App Store metadata   ──→      Creative System       ──→      Creative assets
MoR providers        ←──→     Monetization Engine   ──→      Subscription records
App SDK events       ──→      Event Bus             ──→      Analytics
LLM platforms        ←──→     AI Commerce Layer     ──→      Offer payloads
CRM / Email          ←──→     Lifecycle Engine      ──→      Email sends
```

**Event Bus** is the spine: every user action across web, app, and third-party surfaces emits a normalized event that feeds all agents simultaneously.

---

## 5. Agent Orchestration Layer

The platform's differentiation is that each functional area runs a dedicated agent, and a **meta-agent** coordinates across them to avoid conflicts.

**Meta-agent responsibilities:**
- **Arbitration:** which agent "wins" when multiple want to act on the same user
- **Budget allocation:** rebalance UA spend vs. expected LTV
- **Experiment governance:** ensure valid holdout groups
- **Alert escalation:** surface anomalies to Monetization Console

---

## 6. Compliance & Trust Layer

- **Data residency:** events tagged with user region, routed to appropriate storage
- **Consent signals:** propagated to all agents — no targeting of opted-out users
- **COPPA:** age gate on onboarding; under-13 users excluded from personalization
- **Audit trail:** every agent action logged immutably with actor, timestamp, rationale
- **SOC 2:** access controls, encryption at rest/transit, incident response playbook

---

## 7. Key Design Tensions

| Tension | Trade-off |
|---|---|
| Agent autonomy vs. developer control | Agents act within guardrails; console lets humans narrow or expand the action space |
| Real-time personalization vs. experiment validity | Bandit arms must respect holdout groups; no personalization leaks into control |
| Web checkout vs. App Store | App Store takes 30%; web checkout requires user to trust a new payment surface |
| Offer fatigue | Frequency caps per user — agent tracks how many offers shown in rolling window |
| MoR provider lock-in | Abstraction layer over providers; switching cost should be config, not code |

---

## 8. Phased Build Plan

> **How to read this section:**
> - **Phases are sequential** — each phase's output unlocks the next.
> - **Tracks within a phase are parallel** — each track is an independently assignable workstream.
> - **Tasks within a track are sequential** unless marked `[parallel]`.

---

### Phase 0 — Foundation (3–4 weeks)

All tracks run in parallel. Nothing in Phase 1 can start until Phase 0 checkpoints are met.

#### Track P0-A: Infrastructure & Environments

| Task | Description | Deliverable |
|---|---|---|
| P0-A1 | Cloud provider setup (AWS/GCP), VPC, networking, IAM baseline | Live environments: dev / staging / prod |
| P0-A2 | CI/CD pipeline: lint, test, build, deploy per service | Every repo auto-deploys on merge |
| P0-A3 | Secrets management (Vault or cloud-native), env separation | No secrets in code; rotation policy in place |
| P0-A4 | Observability stack: structured logging, metrics, distributed tracing | Dashboards + alerting baseline live |
| P0-A5 | Database provisioning: Postgres, Redis, ClickHouse/TimescaleDB | All datastores up with backup/restore tested |

#### Track P0-B: Core Data Models

| Task | Description | Deliverable |
|---|---|---|
| P0-B1 | Schema design: App, Team, User, Event, Offer, Subscription, Campaign, Paywall, CreativeAsset | ERD document + reviewed schema |
| P0-B2 | Migration framework setup (Flyway, Alembic) | Schema versioning workflow established |
| P0-B3 | Seed data + fixtures for testing | Dev environment bootstraps with realistic test data |
| P0-B4 | Data access layer (repositories/DAOs) — pure CRUD, no business logic | Tested data layer for all entities |

> ⚠️ **Checkpoint:** Schema must be reviewed and frozen before Phase 1 begins.

#### Track P0-C: Auth & Identity

| Task | Description | Deliverable |
|---|---|---|
| P0-C1 | Developer-side auth (team login, API keys): JWT-based | Teams can sign up, log in, generate API keys |
| P0-C2 | RBAC model: Owner / Admin / Developer / Viewer roles per App | Permission checks middleware |
| P0-C3 | Consumer identity resolution: `device_id`, `email`, `anonymous_id` → unified user record | Identity graph service |
| P0-C4 | SDK API key validation: rate-limited, scoped to App | All SDK calls authenticated |

#### Track P0-D: Event Ingestion Pipeline *(the platform spine)*

| Task | Description | Deliverable |
|---|---|---|
| P0-D1 | Event schema: `{event_type, user_id, app_id, timestamp, properties, session_id, region}` | Event schema doc + validation library |
| P0-D2 | Ingest API: `POST /events` — high-throughput, validates schema, writes to queue | Endpoint handling >10k events/sec |
| P0-D3 | Event queue (Kafka or SQS/SNS): topics per event category | Events durably queued, consumer groups established |
| P0-D4 | Event consumers: fan-out to analytics store, user profile store, real-time state | Events land in all downstream stores within <2s |
| P0-D5 | Event replay capability: reprocess historical events for backfills | Replay pipeline documented and tested |

#### Track P0-E: SDK Skeleton

| Task | Description | Deliverable |
|---|---|---|
| P0-E1 | iOS SDK skeleton: init, identity, event emission | Swift package, publishable to SPM |
| P0-E2 | Android SDK skeleton: init, identity, event emission | Kotlin library, publishable to Maven |
| P0-E3 | Web JS snippet: init, identity, event emission | CDN-hostable `<script>` snippet |
| P0-E4 | SDK integration test harness | Automated tests that validate event emission end-to-end |

#### Track P0-F: Service Architecture

| Task | Description | Deliverable |
|---|---|---|
| P0-F1 | API gateway: routing, auth middleware, rate limiting, versioning (`/v1/`) | Single entry point for all APIs |
| P0-F2 | Service registry + inter-service communication pattern (OpenAPI contracts) | Service mesh or contract-first API specs |
| P0-F3 | Background job framework (Celery, BullMQ, or Temporal) | Workers can run scheduled + triggered jobs reliably |
| P0-F4 | Feature flag system (internal) | Teams can enable/disable features per App without deploy |

---

### Phase 1 — Core Services (5–6 weeks)

All 6 tracks run in parallel. Each requires Phase 0 complete.

#### Track P1-A: Subscription & MoR Layer

| Task | Description | Deliverable | Parallel with |
|---|---|---|---|
| P1-A1 | MoR abstraction interface: `createSubscription`, `cancelSubscription`, `getStatus`, `issueRefund` | Provider-agnostic interface | — |
| P1-A2 | Stripe integration | Stripe can create/cancel subs, handle webhooks | P1-A3, P1-A4 |
| P1-A3 | RevenueCat integration | RevenueCat synced | P1-A2, P1-A4 |
| P1-A4 | Paddle integration | Paddle synced | P1-A2, P1-A3 |
| P1-A5 | Subscription state machine: `trialing → active → past_due → canceled → winback_eligible` | State transitions fire events to pipeline | after A1 |
| P1-A6 | Dunning logic: payment retry schedule (D1, D3, D7 post-failure) | Automated retry with configurable schedule | after A5 |
| P1-A7 | Entitlement service: `hasAccess(user_id, feature)` | Single source of truth for paid access | after A5 |
| P1-A8 | Webhook normalization: normalize all MoR webhooks into standard `subscription.*` events | Unified event format from all providers | after A2–A4 |

#### Track P1-B: Offer Catalog Service

| Task | Description | Deliverable |
|---|---|---|
| P1-B1 | Offer data model: `type`, `price_point`, `currency`, `trial_days`, `copy`, `creative_ref`, `conditions` | Schema migrated and tested |
| P1-B2 | Offer CRUD API: create, update, archive, list offers per App | REST API with validation |
| P1-B3 | Offer template library: pre-built templates by vertical (3–5 per vertical) | Templates for fitness, education, productivity, etc. |
| P1-B4 | Offer conditions engine: `isEligible(offer, user_context) → bool` | Evaluates `segment_match`, `timing_rule`, `region_rule` |
| P1-B5 | Offer versioning: immutable history of offer changes | Offers versioned; experiments reference version IDs |

#### Track P1-C: Web-to-App Funnel

| Task | Description | Deliverable |
|---|---|---|
| P1-C1 | Landing page template engine: parameterized pages per App | Generates unique landing page URLs per campaign |
| P1-C2 | Session tracking: anonymous session ID, UTM capture, scroll/click events | Emits `web_visit`, `cta_click` events |
| P1-C3 | Web checkout flow: email → Stripe payment → issue subscription | Users can pay on web before installing |
| P1-C4 | Deferred deeplink service: generate deeplink with `session_id`; resolves on app first-open | App first-open receives the web session context |
| P1-C5 | Attribution join: link `web_session_id → install → first_open → subscription` | Attribution record stored per user |
| P1-C6 | App Store redirect path: UTM-tagged link + SKAdNetwork (iOS) + referrer API (Android) | Installs from this path are measurable |
| P1-C7 | Funnel drop-off monitoring: alert on threshold breach | Alerting rules + funnel metrics |

#### Track P1-D: Paywall & Experiment Framework

| Task | Description | Deliverable |
|---|---|---|
| P1-D1 | Experiment service: create experiment, assign users to arms, track assignment | `getArm(experiment_id, user_id) → arm` |
| P1-D2 | Statistical significance calculator: automated winner detection | Configurable confidence level |
| P1-D3 | Paywall config API: `getPaywall(user_id, app_id) → {layout, offers, experiment_arm}` | Correct paywall per user |
| P1-D4 | SDK paywall module (iOS): renders paywall from config, fires events | Native paywall UI in iOS SDK |
| P1-D5 | SDK paywall module (Android): same as D4 `[parallel]` | Native paywall UI in Android SDK |
| P1-D6 | Holdout group management: reserve % of traffic as unexposed control | Valid baseline for all experiments |

#### Track P1-E: Monetization Console (Frontend)

| Task | Description | Deliverable |
|---|---|---|
| P1-E1 | Design system + component library | Reusable UI components for all console screens |
| P1-E2 | Auth screens: login, team setup, invite member, API key management | Developers can log in and set up their App |
| P1-E3 | App setup wizard: add App, connect MoR provider, SDK install instructions | New developer onboarded in <15 min |
| P1-E4 | Offer management screen: list, create, edit, archive, view performance | Full offer CRUD in UI |
| P1-E5 | Experiment dashboard: active experiments, arm traffic, CVR per arm, winner callout | Developers manage A/B tests without code |
| P1-E6 | Funnel overview screen: install → trial → paid → retained waterfall | Core KPI view |

#### Track P1-F: Compliance Layer

| Task | Description | Deliverable |
|---|---|---|
| P1-F1 | Audit log service: every write action appended to immutable log with actor + timestamp | `auditLog.write(action, actor, entity, before, after)` |
| P1-F2 | Consent signal propagation: GDPR opt-out respected everywhere | No events for opted-out users |
| P1-F3 | Data residency tagging: events tagged with `region`, routed to regional storage | Events stored in compliant region |
| P1-F4 | COPPA controls: age gate flag; under-13 excluded from all personalization | Personalization engine checks flag before acting |
| P1-F5 | Data retention policies: automated purge jobs per region | Scheduled jobs enforce retention windows |

---

### Phase 2 — Integration & Intelligence (5–6 weeks)

All 6 tracks run in parallel. Requires Phase 1 complete (or at defined checkpoints per track).

#### Track P2-A: Channel Integrations

| Task | Description | Deliverable |
|---|---|---|
| P2-A1 | Channel integration interface: `createCampaign`, `updateBudget`, `pauseCreative`, `pullMetrics` | Provider-agnostic interface |
| P2-A2 | Meta (Facebook/Instagram) integration via Meta Marketing API | Meta campaigns manageable from FloKit |
| P2-A3 | AppLovin integration via Campaigns API `[parallel with A2]` | AppLovin campaigns manageable |
| P2-A4 | Metrics normalization: `{impressions, clicks, installs, cost, revenue}` | Unified campaign metrics table |
| P2-A5 | Campaign builder service: audience spec + creative set + budget → call channel API | `createCampaign(spec) → campaign_id` |
| P2-A6 | Performance pull job: daily metrics from all channels into analytics DB | Campaign performance available in console |

#### Track P2-B: Creative Generation Pipeline

| Task | Description | Deliverable |
|---|---|---|
| P2-B1 | App catalog ingestion: parse App Store URL — screenshots, description, metadata | `App.catalog` populated |
| P2-B2 | Copy generation: LLM prompt chain → headline variants, CTAs, body copy | N copy variants per brief |
| P2-B3 | Image asset generation: channel-appropriate dimensions (1:1, 9:16, 1.91:1) `[parallel with B4]` | Static creative assets |
| P2-B4 | Video creative generation: 15s/30s video variants `[parallel with B3]` | Video assets |
| P2-B5 | Creative composition: overlay copy on visuals, apply brand kit, export per channel spec | Final creative assets ready for upload |
| P2-B6 | Creative library: store all assets with metadata | Searchable asset library |
| P2-B7 | Creative performance tagging: link `creative_id` to campaign metrics | Each asset has a CTR/CVR track record |

#### Track P2-C: Lifecycle Engine

| Task | Description | Deliverable |
|---|---|---|
| P2-C1 | User lifecycle state machine: `anonymous → trial → subscribed → at_risk → churned → winback_eligible` | State machine consuming subscription events |
| P2-C2 | Trigger engine: `{state=trial, days_since_install=3, sessions<2} → fire activation_email` | Triggers evaluate on every user state change |
| P2-C3 | Email template system: activation, nudge, renewal, dunning, win-back; LLM personalizes per segment | Template library with variable substitution |
| P2-C4 | Email sending service: wraps SendGrid/Postmark/SES; handles unsubscribes, bounces | Reliable email delivery with deliverability tracking |
| P2-C5 | Send-time optimization: per-user model of historical open rates by hour | Emails sent at individually optimal times |
| P2-C6 | In-app messaging trigger: push notification or in-app banner via SDK | `sendInAppMessage(user_id, template_id)` |
| P2-C7 | Frequency cap: max N contacts per 7-day rolling window across all channels | No user is over-messaged |

#### Track P2-D: Segment Engine

| Task | Description | Deliverable |
|---|---|---|
| P2-D1 | Segment definition model: rules-based `{country=US AND sessions>3 AND attribution=facebook}` | Segment CRUD API |
| P2-D2 | Segment computation: batch (daily) + real-time update on event receipt | Every user has computed segments |
| P2-D3 | Segment membership API: `getSegments(user_id)` and `getUsersInSegment(segment_id)` | Fast lookup for all downstream consumers |
| P2-D4 | Segment analytics: size, growth, conversion rate per segment | Developers see how each segment behaves |

#### Track P2-E: Console — Intelligence Screens

| Task | Description | Deliverable |
|---|---|---|
| P2-E1 | Campaign management screen: create/edit campaigns, assign creatives, set budget | Acquisition management without leaving console |
| P2-E2 | Creative library screen: browse, preview, tag, assign assets to campaigns | Visual asset management |
| P2-E3 | Lifecycle flow builder: visual editor for trigger → action rules | Non-technical operators configure lifecycle |
| P2-E4 | Segment manager: create, name, preview segments, see membership | Segment management UI |
| P2-E5 | Subscription analytics: MRR, ARR, churn rate, LTV estimates, cohort retention | Core revenue metrics |

#### Track P2-F: Subscription Offer Builder

| Task | Description | Deliverable |
|---|---|---|
| P2-F1 | Vertical pricing heuristics library: baseline trial/annual/monthly pricing per category | Pricing knowledge base |
| P2-F2 | Competitor pricing ingestion: parse competitor pricing from App Store category | Enriched pricing context |
| P2-F3 | Offer generation service: LLM + heuristics → 3–5 offer candidates | Auto-generated offer set ready for A/B test |
| P2-F4 | Offer generation UI: wizard → category → review → approve and push to catalog | Developers get a starting offer set in minutes |

---

### Phase 3 — AI Agent Layer (5–7 weeks)

All tracks run in parallel. **Requires Phase 2 producing real event data.**

#### Track P3-A: Offer Scoring Agent *(Core AI)*

| Task | Description | Deliverable |
|---|---|---|
| P3-A1 | Feature vector definition: `[segment, channel, time_in_app, session_count, offer_history, recency]` | Feature spec document |
| P3-A2 | Training data pipeline: pull historical `(user_context, offer_shown, outcome)` tuples | Labeled dataset for initial model training |
| P3-A3 | Contextual bandit model: train initial model | `score(user_context) → [{offer_id, score}]` deployed |
| P3-A4 | Offer dispatch service: call scorer, select offer, respect guardrails, dispatch to surface | `dispatch(user_id, surface) → offer` |
| P3-A5 | Outcome feedback loop: conversion/dismiss events fed back to retrain model daily | Model improves over time |
| P3-A6 | Guardrail enforcement: min/max price, region restrictions, experiment arm locks before dispatch | No invalid offer ever shown |

#### Track P3-B: Meta-Agent Orchestration

| Task | Description | Deliverable |
|---|---|---|
| P3-B1 | Agent registry: catalog of all active agents per App, action domains, current status | Central registry of running agents |
| P3-B2 | Action arbitration: priority rules when multiple agents want to act on same user | No conflicting actions reach the same user |
| P3-B3 | Budget meta-allocation: monitor UA spend vs. projected LTV → recommend rebalancing | Weekly budget recommendation in console |
| P3-B4 | Anomaly detection: monitors key metrics; fires alerts on threshold breach within <1h | Alert within <1h of meaningful regression |
| P3-B5 | Agent action log: every agent decision recorded with rationale, inputs used, outcome | Full audit trail for every autonomous action |

#### Track P3-C: AI Commerce & MCP Integration

| Task | Description | Deliverable |
|---|---|---|
| P3-C1 | Offer surface registry: enum of all surfaces and their rendering constraints | Surface specs document |
| P3-C2 | One-tap checkout service: resolve payment method, charge, provision entitlement | `checkout(user_id, offer_id) → {status, entitlement}` |
| P3-C3 | Chatbot webhook: endpoint accepts `{user_context}` → returns `{offer_payload}` | Chatbot can surface offers via webhook |
| P3-C4 | MCP tool implementation: FloKit exposes `get_offer(user_context)` as MCP tool | MCP tool spec + server deployed |
| P3-C5 | Email offer card: rendered HTML offer block with one-click checkout link (tokenized, short-lived) | Offer in email clicks through to one-tap checkout |
| P3-C6 | Surface-aware offer selection: agent adapts offer content to surface constraints | Each surface gets appropriately formatted offer |

#### Track P3-D: Creative Iteration Agent

| Task | Description | Deliverable |
|---|---|---|
| P3-D1 | Performance evaluation job: daily CTR, CVR, cost-per-install vs. benchmark | Underperformer flag on each creative |
| P3-D2 | Creative retirement: automatically pause creatives below threshold | Underperformers stop spending |
| P3-D3 | Replacement generation: when creative retired, agent triggers new variant generation | New variants auto-queued |
| P3-D4 | Creative learning: extract what worked from top performers → seed next generation | Generation prompts informed by performance data |

#### Track P3-E: Agent Console UI

| Task | Description | Deliverable |
|---|---|---|
| P3-E1 | Agent activity feed: real-time stream of agent decisions with context and outcome | Operators see what agents are doing live |
| P3-E2 | Guardrail editor: UI to set min/max price, max discount, region exclusions, frequency caps | Non-engineer operators can constrain agents |
| P3-E3 | Override controls: force specific offer to 100% traffic, pause agent category | One-click human override |
| P3-E4 | Recommendation surface: agent surfaces high-impact suggestions → operator approves | Human-in-the-loop for high-impact decisions |

---

### Phase 4 — Advanced Commerce & Growth (4–5 weeks)

All tracks run in parallel. Requires Phase 3 agents operating with real data.

#### Track P4-A: UA Revenue-Share Financing

| Task | Description | Deliverable |
|---|---|---|
| P4-A1 | LTV prediction model: predict D30/D90 LTV from D3/D7 cohort signals | `predictLTV(cohort_features) → {ltv_estimate, confidence}` |
| P4-A2 | Financing application flow: developer requests UA budget; system assesses App LTV profile; approves/declines | Approval decision with financing terms |
| P4-A3 | Repayment tracking: as cohort revenue is recognized, calculate FloKit's share | Revenue attribution and repayment ledger |
| P4-A4 | Risk controls: per-App exposure limits; cohort performance gates (if D7 LTV < floor → pause spend) | Automated risk management |

#### Track P4-B: ASO & AI Surface Discovery

| Task | Description | Deliverable |
|---|---|---|
| P4-B1 | ASO analysis: parse App Store page, keyword rankings, ratings vs. category top 10 | ASO health score + gap analysis per App |
| P4-B2 | ASO recommendation agent: suggests keyword, screenshot, and description changes | Prioritized ASO action list |
| P4-B3 | AI surface discovery: monitor LLM/AI surfaces for category keywords | Alert when App category surfaced in AI-native results |
| P4-B4 | AI surface optimization: generate App descriptions optimized for AI retrieval | App appears in relevant AI surface results |

#### Track P4-C: Advanced Analytics & LTV

| Task | Description | Deliverable |
|---|---|---|
| P4-C1 | Cohort analysis engine: group users by install week × channel × segment | Cohort table in analytics DB |
| P4-C2 | LTV projection dashboard: D7 → D30 → D90 → D365 LTV curves with confidence bands | Developers see predicted revenue trajectory |
| P4-C3 | Payback period calculator: CAC by channel + LTV curve → days to payback | Payback period per channel in console |
| P4-C4 | Win-back campaign analytics: incremental revenue vs. organic return rate | True incrementality measurement |

#### Track P4-D: Win-Back & Churn Recovery

| Task | Description | Deliverable |
|---|---|---|
| P4-D1 | Churn prediction model: predict probability of churning in next 14 days | `predictChurn(user_id) → {probability, top_signals}` |
| P4-D2 | Proactive retention flow: for high-risk users, agent selects intervention before churning | At-risk users receive targeted retention action |
| P4-D3 | Win-back campaign engine: scheduled sequence of escalating discount offers over 60 days | Win-back sequence with incrementally better offers |
| P4-D4 | Pause subscription option: 30/60-day pause alternative to hard cancel | Pause option surfaced in cancellation flow |

---

## 9. Critical Path

The chain that nothing can bypass:

```
P0-D  Event Pipeline
  ↓
P1-A  Subscription State Machine
  ↓
P2-C  Lifecycle State Machine
  ↓
P2-D  Segment Engine
  ↓
P3-A  Training Data Pipeline
  ↓
P3-A  Offer Scoring Agent
  ↓
P3-B  Meta-Agent Arbitration
```

Every other track is parallelizable around this spine. Staff the critical path with your strongest engineers and shield it from scope creep.

---

## 10. Phase Summary

| Phase | Name | Duration | Tracks | Key Unlock |
|---|---|---|---|---|
| **0** | Foundation | 3–4 weeks | 6 | Event pipeline + data models live |
| **1** | Core Services | 5–6 weeks | 6 | MoR, paywall, web funnel, console skeleton |
| **2** | Integration & Intelligence | 5–6 weeks | 6 | Channels connected, lifecycle running, data flowing |
| **3** | AI Agent Layer | 5–7 weeks | 5 | Agents operating autonomously within guardrails |
| **4** | Advanced Commerce & Growth | 4–5 weeks | 4 | Financing, ASO, churn prediction, AI surfaces |
| **Total** | — | **~22–28 weeks** | **27** | Full platform |

---

## 11. Staffing Allocation

| Role | Phase 0 | Phase 1 | Phase 2 | Phase 3 | Phase 4 |
|---|---|---|---|---|---|
| Platform / Infra | ●●● | ● | ● | ● | ● |
| Backend Services | ●● | ●●● | ●●● | ●● | ●● |
| AI / ML | — | — | ● | ●●● | ●● |
| Frontend / Console | — | ●● | ●● | ●● | ● |
| Mobile SDK | ● | ●● | ● | — | — |
| Compliance | — | ● | ● | ● | — |

*Each ● = 1 engineer headcount.*

**Role responsibilities:**
- **Platform / Infra:** Cloud setup, CI/CD, observability, databases, event pipeline, API gateway
- **Backend Services:** MoR layer, offer catalog, web-to-app funnel, lifecycle engine, segment engine, channel integrations
- **AI / ML:** Contextual bandit model, training pipelines, LTV prediction, churn prediction, anomaly detection
- **Frontend / Console:** Monetization console, campaign management, creative library, agent console, lifecycle flow builder
- **Mobile SDK:** iOS and Android SDK — identity, events, paywall modules
- **Compliance:** Audit logging, consent propagation, data residency, COPPA controls, retention policies
