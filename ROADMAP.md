# FloKit.AI — Product Roadmap & Technical Feature Specification

**Internal Engineering Document**
**Date:** June 2026

---

## Table of Contents

1. [Platform Vision](#1-platform-vision)
2. [Architecture Overview](#2-architecture-overview)
3. [Feature Domains](#3-feature-domains)
   - [Brand & App Store Intelligence](#31-brand--app-store-intelligence)
   - [Creative Engine](#32-creative-engine)
   - [User Acquisition (UA) Automation](#33-user-acquisition-ua-automation)
   - [Virality & Social Feedback](#34-virality--social-feedback)
   - [Organic Growth Engine](#35-organic-growth-engine)
   - [Activation & Conversion Pipeline](#36-activation--conversion-pipeline)
   - [Retention & Engagement Layer](#37-retention--engagement-layer)
   - [Monetization Infrastructure](#38-monetization-infrastructure)
   - [Ops & Support Automation](#39-ops--support-automation)
   - [Unified Data Layer](#310-unified-data-layer)
   - [Predictive ML Models](#311-predictive-ml-models)
4. [Agentic Autopilot Layer](#4-agentic-autopilot-layer)
5. [Integrations & MoR Strategy](#5-integrations--mor-strategy)
6. [Phased Roadmap](#6-phased-roadmap)
7. [Target Customer Profile](#7-target-customer-profile)

---

## 1. Platform Vision

FloKit.AI is an **agentic growth infrastructure platform** for consumer subscription apps. The problem it solves: growth teams at subscription apps spend most of their time on manual, reactive loops — reviewing dashboards, setting up A/B tests, adjusting bids, writing lifecycle emails. These steps can be modeled as feedback loops and delegated to autonomous agents that observe signals and act continuously.

**Core proposition:** Give FloKit.AI read access to your existing data sources (App Store Connect, ad networks, MMP, payment provider, CRM, website analytics), and the system generates, executes, and iterates your entire growth stack — creatives, paywalls, UA campaigns, email sequences, referral flows — with human approval gates at configurable thresholds.

**Competitive positioning:**
- RevenueCat / Adapty — payment infrastructure only; no acquisition or lifecycle
- Paddle — MoR only; no growth automation
- Appsflyer / Singular — attribution only; no action layer
- ZyG — comparable vision for e-com; FloKit.AI is the equivalent for mobile subscription apps

**Three rails (mirroring the FLOKITAI_BUILD_PLAN architecture):**
- **Acquisition rail** — ad creative generation → campaign deployment → bidding optimization → attribution
- **Monetization rail** — paywall construction → offer personalization → subscription lifecycle
- **Lifecycle rail** — onboarding → habit formation → churn prediction → win-back

All three rails share one event bus and are orchestrated by an AI agent layer, not manual operator workflows.

---

## 2. Architecture Overview

```
┌───────────────────────────────────────────────────────────┐
│                   Data Ingestion Layer                     │
│  App Store Connect │ MMPs │ Ad Networks │ Payments │ CRM  │
└────────────────────────┬──────────────────────────────────┘
                         │  Normalized Event Stream
┌────────────────────────▼──────────────────────────────────┐
│                  Unified Data Layer                        │
│   Event Store │ Identity Graph │ Cohort Engine │ A/B Core │
└────────────────────────┬──────────────────────────────────┘
                         │
           ┌─────────────┼─────────────┐
           ▼             ▼             ▼
    ┌──────────┐  ┌──────────┐  ┌───────────┐
    │  Acq.    │  │  Monet.  │  │ Lifecycle │
    │  Agent   │  │  Agent   │  │  Agent    │
    └──────────┘  └──────────┘  └───────────┘
           │             │             │
           └─────────────┼─────────────┘
                         ▼
               ┌─────────────────┐
               │  Autopilot Hub  │   ← human review / approval gates
               │  (Hypothesis    │
               │   Pool + Bench) │
               └─────────────────┘
                         │
              ┌──────────┼──────────┐
              ▼          ▼          ▼
         Ad Networks  Paywalls  CRM/Push/Email
```

### Key technical layers

| Layer | Technology choices | Responsibility |
|---|---|---|
| Event bus | Kafka / Redpanda | All behavioral events, real-time + batch |
| Data warehouse | ClickHouse or BigQuery | Cohort queries, funnel analysis, ML feature store |
| Identity graph | Custom service + PostgreSQL | Device ↔ user ↔ subscription ↔ attribution linkage |
| ML serving | FastAPI + ONNX runtime | Low-latency scoring (<20 ms p99) |
| Agent orchestration | Internal DAG engine or Temporal | Campaign triggers, lifecycle sequences, test evaluation |
| Creative pipeline | FFmpeg + Stable Diffusion / Runway API | Asset generation and transformation |
| A/B / bandits | In-house experimentation layer | Allocation, stats engine, auto-stop |
| MCP layer | Model Context Protocol endpoints | Embeds offers/paywalls into LLM/chatbot contexts |

---

## 3. Feature Domains

### 3.1 Brand & App Store Intelligence

#### App Store Optimization (ASO)

**Technical spec:**
- Scrapes App Store and Google Play metadata daily via public APIs and headless browser automation
- Keyword ranking tracker: stores historical rank data per keyword per country in time-series DB (ClickHouse or TimescaleDB)
- Competitive keyword gap analysis: TF-IDF style scoring against top-10 competitors in category
- Metadata generator: fine-tuned LLM produces title/subtitle/description variants constrained to character limits and keyword density targets
- Screenshot and icon scoring: CLIP embeddings of top-ranked app screenshots → cosine similarity ranking against category median

**Outputs:** Ranked keyword list, metadata variants ready for A/B test, visual asset annotations.

#### Store Page A/B Testing

- Deep-links Android users into Custom Product Pages (CPP) and iOS users into Product Page Optimization (PPO) experiments
- Assigns traffic allocation via experiment UUID in deep-link parameter
- Measures conversion rate (store page → install) using SKAdNetwork attribution on iOS and referrer on Android
- Bayesian stopping rule: test terminates when posterior probability of winner exceeds configurable threshold (default 95%)

#### Landing Page Optimization

- Injects JavaScript snippet on app's web-to-app landing pages
- Captures UTM, creative, ad set, and campaign params
- Routes variants via CDN edge config (Cloudflare Workers or Lambda@Edge) — no deploy required per test
- Measures: click-to-install rate, time-on-page, scroll depth, form completion

#### Review & Rating Management

- Polls App Store Connect / Play Console reviews API every 6 hours
- Sentiment classifier (fine-tuned RoBERTa) categorizes each review by topic (crash, UX, pricing, feature request)
- In-app prompt engine: surfaces review request only to users with ≥ 3 sessions + positive engagement signal, suppressed for users with support tickets or payment failures
- Auto-drafts developer reply templates per sentiment cluster, routed for human approval before publish

---

### 3.2 Creative Engine

#### Creative Testing Infrastructure

- Creative variants stored in asset catalog with structured metadata: `hook_type`, `format` (video/static/UGC), `audience_tag`, `platform`, `duration_s`
- Launches creatives as new ad set variants within existing campaigns to isolate creative signal
- Creative performance index = composite score of: CPI, D7 retention, D30 subscription rate, LTV projection
- Automated deprecation: creatives below bottom quartile on composite score after 3× impression threshold are paused

#### AI Creative Generation

**Pipeline:**
1. Brief generation — LLM extracts top-performing hooks from existing creative metadata + review sentiment signals
2. Script writing — GPT-4 or Claude generates video scripts in configurable formats: testimonial, hook-hold-CTA, tutorial, UGC-style
3. Visual generation — Stable Diffusion / DALL-E 3 for static assets; Runway / Pika for video clips
4. Asset assembly — FFmpeg compositing: overlays, captions (via Whisper), music bed, end-card CTA
5. Platform sizing — automatic reformat to 9:16 (TikTok/Reels/Shorts), 1:1 (feed), 16:9 (YouTube)

**UGC briefs:** Structured JSON output: `{hook, key_claim, visual_direction, CTA, target_emotion}` — sent to creator management platform or DM outreach sequences.

#### Creative Performance Scoring

| Signal | Weight | Source |
|---|---|---|
| CPI (channel-normalized) | 25% | Ad network APIs |
| D7 retention | 20% | MMP → event store |
| Trial start rate | 20% | Payment provider |
| D30 subscription rate | 20% | Payment provider |
| LTV projection (model) | 15% | ML scoring service |

Scoring runs daily; output stored in creative asset catalog. Leaderboard surfaced in Autopilot Hub.

#### Hook & Message Analysis

- NLP pipeline over creative scripts + review text + app store descriptions
- Extracts: `emotion_trigger` (fear, aspiration, social proof, urgency), `claim_type` (outcome, process, comparison, social), `specificity_score`
- Correlation matrix: hook attribute × downstream conversion metric
- Actionable output: ranked list of hook patterns with statistical lift estimates

---

### 3.3 User Acquisition (UA) Automation

#### Paid Acquisition Optimization

**Supported networks (API-connected):**
- Meta (Marketing API v19+)
- TikTok Ads API
- Google UAC / DV360
- Apple Search Ads (Campaign Management API)
- AppLovin MAX

**Bidding strategy:**
- Reads target LTV/CAC ratio per app (configurable per team)
- Adjusts target CPA bids daily based on rolling 7-day cohort LTV estimates
- Pause logic: ad set with >500 installs and LTV/CAC < 0.6 after 14 days → auto-pause + alert
- Budget reallocation: surplus budget from paused ad sets redistributed to top performers proportionally

#### Budget Allocation Engine

- Inputs: channel-level LTV/CAC, creative score, audience saturation signal (frequency), projected scale ceiling
- Solves constrained optimization: maximize projected LTV subject to total budget constraint and minimum spend floors per channel
- Runs nightly; proposes reallocation; executes after approval gate or auto-applies if delta < configurable threshold (default: ±15%)

#### Audience Segmentation

- Builds lookalike seed audiences from: high-LTV subscriber cohort, D30 retained users, converted trial users
- Segments pushed to Meta / TikTok / Google via Customer Audience API (hashed emails/device IDs)
- Suppression lists: churned users, existing subscribers, refund requestors — pushed as negative audiences
- Behavioral segments enriched from in-app event data via MMP integration

#### Attribution & Incrementality

- MMP integration: AppsFlyer, Adjust, Singular (OAuth-based data pull)
- Incrementality: geo-holdout experiments — split markets by DMA/country into test (ads running) and holdout (ads paused); measure organic baseline vs. incremental lift
- Model-based attribution: Shapley value allocation across touchpoints for multi-touch journeys
- View-through attribution deduplication: configurable window (default 1 day) with cross-device graph fallback

---

### 3.4 Virality & Social Feedback

#### Referral Loops

**Technical implementation:**
- Referral code generation: `{app_id}_{user_id_hash}_{campaign_slug}` — stored in DB, resolved on app open
- Branch.io or custom deep-link service maps referral codes to install attribution
- Reward engine: event-driven microservice triggers reward (credit, unlock, discount) when referral condition is met (install + activation event)
- Anti-abuse: device fingerprint deduplication + velocity rules (max N referrals per device per 24 h)

#### Invite Flows

- SDK module (iOS + Android) exposes: `showInviteSheet(trigger: InviteTrigger)` 
- Channels: native share sheet (iOS), WhatsApp direct-share intent, SMS, email, copy-link
- Contact-picker permission flow with soft ask pattern before OS prompt
- Deep-links carry: `referrer_id`, `campaign`, `landing_variant`, resolved at first open

#### Sharing Templates

- Template system: `ShareTemplate { type, imageURL, caption, deepLink, fallbackURL }`
- Dynamic OG tag generation: server-side renders preview images per share context (score, achievement, result) using canvas-based renderer
- Templates triggered by in-app events: level complete, streak milestone, quiz result, subscription upsell dismiss

#### Viral Loop Analytics

| Metric | Definition | Storage |
|---|---|---|
| K-factor | installs from referrals / total installs (rolling 7d) | ClickHouse daily aggregates |
| Invite rate | users who sent ≥1 invite / DAU | Event store |
| Referral conversion | installs from referral that reached activation event | Cohort engine |
| Viral retention | D30 of referred users vs. organic cohort | Cohort comparison |

---

### 3.5 Organic Growth Engine

#### SEO / LLM Discovery

- Programmatic SEO: templated content pages generated for app use-cases, populated by LLM, indexed via sitemap auto-generation
- LLM discoverability: structured JSON-LD schema markup (`SoftwareApplication`, `FAQPage`) on landing pages
- Keyword clustering: groups long-tail keywords by semantic intent cluster; assigns to page templates
- Perplexity / ChatGPT citation monitoring: tracks mentions of app name and category terms in AI search results via API polling

#### Content Engine

- Template types: `{use_case_page, comparison_page, how_to_guide, feature_story, integration_page}`
- Content generation pipeline: keyword intent → LLM draft → SEO scoring (readability, keyword density, internal link density) → human review queue → publish
- CMS integration: Webflow / Contentful via API; auto-publish or draft based on review gate config
- Scaling target: 50–200 indexed pages per quarter per app on platform

#### Creator / Influencer Tracking

- Unique UTM + custom deep-link per creator campaign
- Downstream metrics tracked: installs, activations, trial starts, subscriptions attributed to each creator code
- Coupon code redemption tracking via payment provider webhook
- LTV-per-creator metric: cohort LTV of users attributed to each creator vs. paid channel baseline

---

### 3.6 Activation & Conversion Pipeline

#### Onboarding Optimization

**Technical approach:**
- Onboarding defined as a DAG of screens + events in the FloKit SDK
- Each screen registers: `screen_id`, `skip_allowed`, `back_allowed`, `completion_event`
- Funnel constructed automatically from event stream; drop-off identified at each node
- A/B testing at screen level: variant assignment stored in user profile, consistent across sessions

#### Personalized Onboarding

- At install, collects: UTM source, creative tag, country, device type, OS version
- Feeds into onboarding variant selector model: maps `(attribution_source, country, device_tier)` → `onboarding_flow_id`
- Post-signup: intent quiz responses stored as `user.persona_tags[]`; personalization applied to paywall, notification copy, and email sequences
- Example flows: `gaming_casual`, `gaming_competitive`, `productivity_power_user`, `wellness_goal_seeker`

#### Funnel Analytics

- Canonical funnel: `install → first_open → onboarding_start → activation_event → trial_start → subscription → renewal`
- Each step stored as timestamped event; funnel view computed in ClickHouse with user-level JOIN
- Segmentable by: channel, country, creative, persona, cohort week, device tier
- Automatic anomaly detection: funnel step drop-off deviation >15% vs. 7-day average triggers alert

---

### 3.7 Retention & Engagement Layer

#### Push Notifications

**Architecture:**
- Notification microservice: receives trigger events from agent, resolves user FCM/APNs token, applies frequency cap, sends via Firebase or OneSignal
- Frequency cap: per-user configurable (`max_per_day`, `quiet_hours`, `channel_priority`)
- Personalization: notification copy templated with `{{user.first_name}}`, `{{user.streak}}`, `{{user.last_used_feature}}` — populated at send time
- ML-based send time: per-user optimal send hour estimated from historical open-time distribution
- Deep-link routing: all notifications carry `screen_id` + `action_params` for in-app destination

#### Email / SMS / CRM

- Lifecycle sequences modeled as state machines: each user has a `lifecycle_state` (`new`, `onboarded`, `trial`, `subscribed`, `at_risk`, `churned`, `winback`)
- State transitions trigger sequence enrollment/exit
- Email provider: Customer.io, Braze, or Klaviyo (via API abstraction layer — swap provider without code change)
- SMS: Twilio with opt-in management per country (GDPR / TCPA compliant)
- CRM sync: bidirectional sync to HubSpot / Intercom for support and sales workflows

#### Habit Formation Loops

- Streak tracking service: stores `{user_id, streak_type, current_count, last_activity_ts, grace_period_expires_ts}`
- Grace period logic: configurable per app (default: 24 h); grace uses a notification trigger before breaking streak
- Reward engine: event-driven; configured rules like `on: streak=7 → action: grant_feature_unlock`
- Goal-setting: user-set targets stored in profile; progress notifications triggered at 25/50/75/100%

#### Churn Detection

- Feature vector per user (computed daily): `{days_since_last_open, session_frequency_7d_vs_30d, notification_open_rate_trend, payment_status, support_ticket_open, trial_days_remaining}`
- Gradient Boosted Trees (XGBoost/LightGBM) or LSTM sequence model
- Churn risk score 0–1 stored in user profile; thresholds: `>0.6 = at_risk`, `>0.85 = critical`
- Feeds into lifecycle agent: `at_risk` → enroll in retention campaign; `critical` → offer winback discount

---

### 3.8 Monetization Infrastructure

#### Subscription Optimization

**Payment providers (abstraction layer):**
- RevenueCat (primary — handles Apple/Google billing)
- Paddle (web-to-app + web subscriptions, MoR)
- Stripe (direct web billing fallback)

All events normalized to FloKit's internal `subscription_event` schema:
```json
{
  "user_id": "...",
  "event_type": "trial_started | converted | renewed | cancelled | refunded | billing_retry",
  "product_id": "...",
  "price_usd_cents": 999,
  "currency": "USD",
  "provider": "revenuecat | paddle | stripe",
  "timestamp": "ISO8601"
}
```

#### Paywall Testing

- Paywall defined as JSON config: `{layout, offers[], cta_copy, background, social_proof_module, urgency_timer}`
- Variants stored in paywall config service; active variant resolved per user at render time (consistent hashing)
- Metrics tracked per variant: `impression → tap_offer → trial_start → convert_paid` with statistical significance computed in real time
- Auto-winner: Bayesian model declares winner at 95% confidence; champion config promoted; experiment archived

#### Dynamic Pricing

- Geo-pricing: price point matrix per country tier (Tier 1/2/3) stored in offer catalog; applied automatically at paywall render
- Segment-based pricing: model predicts willingness-to-pay from `{country, session_depth, engagement_score, attribution_channel}` → maps to price cohort
- Guardrails: pricing changes always A/B tested before full rollout; min floor price enforced per SKU per region

#### MCP Offers Layer

- FloKit exposes a Model Context Protocol (MCP) server with tool endpoints:
  - `get_user_offer(user_id)` → returns personalized offer JSON
  - `record_offer_interaction(user_id, offer_id, action)` → logs accept/reject/dismiss
  - `get_paywall_config(user_id, context)` → returns full paywall variant config
- Enables AI agents, chatbots, and conversational surfaces (in-app assistant, WhatsApp bot, web chat) to present and close subscription offers natively within conversation context
- Direct purchase flow: user can complete trial start or subscription upgrade via agent conversation without leaving chat context

#### Revenue Cohort Analysis

- Cohort defined by `first_install_week` × `channel` × `country` × `creative_tag`
- Revenue metrics per cohort: `d1_revenue`, `d7_revenue`, `d30_revenue`, `ltv_90`, `refund_rate`, `renewal_rate`
- Payback period calculator: CAC (from ad spend) ÷ daily ARPU projection → days-to-payback per acquisition cohort
- Exported to BI tool or surfaced in FloKit dashboard

---

### 3.9 Ops & Support Automation

#### Customer Support Automation

- LLM-powered support agent (Claude / GPT-4 Turbo) with RAG over: FAQ docs, past support tickets, product changelogs
- Handles Tier 1: "how do I cancel", "I was charged twice", "feature not working" → automated resolution + refund initiation if policy matches
- Escalation routing: confidence < threshold or `intent: billing_dispute_amount > $50` → routes to human agent with full context summary
- Support platform integration: Intercom, Zendesk, or Freshdesk via API; creates/updates tickets

#### Refund / Cancellation Insights

- Cancellation survey: SDK modal on cancel CTA captures exit reason (`{too_expensive, not_useful, technical_issue, found_alternative, other}`)
- Webhook listener on payment provider events: `cancellation_at_period_end`, `immediate_cancel`, `refund_requested`
- Exit intent model: on cancel intent, evaluate user segment + churn score → decide whether to show save offer (discount, pause subscription, downgrade)
- Cohort analysis: refund rate by channel/creative/country/paywall variant → feeds back into acquisition optimization

#### Fraud & Abuse Detection

- Rules engine + ML hybrid:
  - Device fingerprint velocity: >N accounts per device per day → flag
  - Referral abuse: referral + install + referral_reward claimed from same subnet or device cluster → hold reward
  - Payment abuse: multiple failed cards per user + chargeback rate > threshold → block
- IP reputation scoring via MaxMind GeoIP + internal velocity counters
- Fraud events published to event bus → downstream suppression of ad audiences and lifecycle sequences

#### App Health Monitoring

- FloKit SDK exports: crash rate (via Firebase Crashlytics integration), ANR rate, failed payment events, deep-link resolution failures
- Metrics forwarded to monitoring stack (Datadog / Grafana)
- Alerting rules: crash rate >0.5% on version → alert engineering; payment failure spike >5% above baseline → alert growth + engineering
- Correlated with funnel metrics: if crash rate spike correlates with funnel drop at specific step → linked alert includes affected funnel node

---

### 3.10 Unified Data Layer

#### Event Tracking

**FloKit SDK (iOS + Android + Web):**
- Auto-captures: `app_open`, `screen_view`, `first_open`, `push_received`, `push_opened`, `deep_link_opened`
- Manual tracking: `FloKit.track("event_name", properties: {})` — typed schema enforced on SDK side
- Events batched (50 events or 30 s, whichever first) and sent to ingestion endpoint with retry + local persistence on failure
- Server-side events: payment events, backend triggers posted to `/events` API

**Event schema (canonical):**
```json
{
  "event_id": "uuid",
  "user_id": "...",
  "anonymous_id": "...",
  "event": "event_name",
  "properties": {},
  "context": { "device", "os", "app_version", "country", "ip" },
  "timestamp": "ISO8601",
  "sent_at": "ISO8601"
}
```

**Ingestion pipeline:** SDK → HTTPS → API gateway → Kafka topic → stream processors (Flink or ksqlDB) → ClickHouse (analytics) + PostgreSQL (operational) + S3 (raw archive)

#### User Identity Resolution

- **Anonymous phase:** `anonymous_id` = UUID generated at first app open, persisted in Keychain/Keystore
- **Identity merge:** On signup/login, `alias(anonymous_id → user_id)` call creates merge record; all historical events rewritten to `user_id` in batch
- **Cross-device:** email-based matching across devices; device graph stored in identity service
- **Attribution join:** MMP-provided `attribution_click_id` (IDFA / GCLID / FBCLID) joined to user profile at install time

#### Cohort Engine

- Cohort types:
  - **Acquisition cohort:** first install week × channel × country
  - **Behavioral cohort:** users who performed event X within N days of install
  - **Monetization cohort:** trial starts, converters, renewers, churners — by calendar week
  - **ML cohort:** users assigned to same churn risk / LTV bucket by model
- Cohort membership stored as `{user_id, cohort_id, cohort_type, enrolled_at}` in PostgreSQL; queried in ClickHouse for analytics
- Cohort builder UI: drag-drop conditions → generates SQL-equivalent query → materializes cohort for A/B seed audiences, CRM segments, or lookalike upload

#### Experimentation Layer

- Experiment types: `ab_test` (50/50 or multi-arm), `multivariate`, `feature_flag`, `holdout`
- Assignment: deterministic hashing of `(user_id, experiment_id)` → bucket → variant; no server call needed at evaluation time
- CUPED variance reduction applied on primary metrics before significance computation
- Guardrail metrics: each experiment defines mandatory guardrails (e.g., crash rate, payment failure rate) — auto-pauses if guardrail breached
- Results export: per-experiment results stored in data warehouse; surfaced in Autopilot Hub with interpretation guide

---

### 3.11 Predictive ML Models

All models follow the same serving pattern:
1. Feature vector assembled from event store + user profile at request time
2. ONNX model loaded in FastAPI serving container
3. Response cached per user with configurable TTL
4. Predictions logged with input features for monitoring and retraining

#### LTV Prediction

- **Training data:** cohort revenue outcomes at d7, d30, d90 linked back to install-day features
- **Features:** `{attribution_channel, creative_tag, country, device_tier, onboarding_completion_rate, d1_session_count, d1_events, trial_started_flag, persona_tag}`
- **Model:** Gradient Boosted Trees (LightGBM) with survival model component for long-horizon revenue
- **Output:** `ltv_p50_90d`, `ltv_p90_90d` — used for bid optimization and audience seeding
- **Retraining:** weekly, triggered by data freshness check

#### Churn Prediction

- **Features:** (see §3.7 Churn Detection above) + subscription tenure, payment history, session depth trend
- **Label:** `churned = 1` if no app open within 30 days (for trial users: no conversion within trial window)
- **Model:** XGBoost classifier; threshold tuned per app to balance precision/recall based on intervention cost
- **Action trigger:** score written to user profile; lifecycle agent polls on state change

#### Conversion Propensity

- **Scope:** predicts probability of trial-to-paid conversion within 7 days of trial start
- **Features:** `{trial_day, sessions_in_trial, key_feature_activations, push_open_rate, paywall_views, offer_interacted}`
- **Output:** per-user conversion probability; users below threshold at trial day 3 enter early rescue sequence (targeted discount offer or feature tutorial)

#### Next Best Action (NBA)

- **Inputs:** user state, lifecycle stage, last action, predicted churn risk, predicted LTV, current experiment assignments
- **Action space:** `{send_push, send_email, show_in_app_modal, show_paywall, trigger_discount_offer, do_nothing}`
- **Model:** contextual bandit with Thompson sampling; action space constrained by frequency caps and eligibility rules
- **Retraining:** daily batch update + real-time reward signal processing

#### Creative-to-LTV Prediction

- **Goal:** at creative launch time, predict downstream LTV quality of users it will acquire, before sufficient revenue data exists
- **Method:** representation learning — creative embedding (CLIP image + text embedding of script) → linear head trained on `(creative_embedding → d30_ltv_cohort_average)` using historical creative performance data
- **Use case:** before launching a new creative concept, score it against the trained head to estimate LTV quality; deprioritize concepts predicted to attract low-LTV users

---

## 4. Agentic Autopilot Layer

The Autopilot is the top-level orchestration agent that owns the hypothesis pool and closes the optimization loop across all three rails.

### Autopilot Hub

**Hypothesis pool:** All pending experiments organized as `Hypothesis { id, domain, description, rationale, data_source, expected_lift, priority_score, status, experiment_id? }`

**Sources of hypotheses:**
- Automated analysis agents (benchmark agent, funnel anomaly detector, cohort comparison agent)
- Creative scoring signals (new hook patterns with no current test)
- Paywall analysis (screenshot upload → benchmark → suggested variants)
- Human input from growth team

**Priority scoring:** `priority = expected_lift × confidence × (1 / estimated_test_duration_days)`

### Autopilot Agents

| Agent | Trigger | Action |
|---|---|---|
| **Benchmark Agent** | Weekly or on-demand | Compares KPIs vs. category benchmarks; generates hypothesis list |
| **Funnel Anomaly Agent** | Daily | Detects drop-off anomalies; hypothesizes causes; proposes tests |
| **Creative Refresh Agent** | When top creative >30d old or CPI degrading | Generates new creative brief based on top-hook analysis |
| **Bid Optimization Agent** | Daily | Recalculates target CPAs; proposes budget reallocation |
| **Paywall Analysis Agent** | On paywall screenshot upload or weekly | Benchmarks paywall vs. category; generates variant config |
| **Win-back Agent** | On churn event batch | Segments churned users; launches personalized win-back sequence |

### Human-in-the-Loop Gates

- **Auto-approve:** A/B test launch where budget delta < $X/day, copy changes, notification copy variants
- **Requires approval:** creative launch to new audience, budget reallocation >15%, paywall config change, pricing change
- **Always requires approval:** new campaign launch, audience targeting changes, any price point change in Tier 1 markets

### Autopilot Paywall Analysis (Screenshot Feature)

1. User uploads paywall screenshot
2. Vision model (GPT-4o / Claude 3.5) extracts: layout type, offer count, CTA text, trust signals, urgency elements, price anchoring
3. Benchmark lookup: category + subcategory paywall patterns from internal database of top-100 app paywalls
4. Gap analysis: identifies missing elements vs. benchmark (e.g., "no social proof", "single-price anchoring", "no trial CTA")
5. Auto-generates 2–3 variant configs in FloKit paywall JSON format
6. Presents to user in Autopilot Hub with rationale; one-click launches A/B test

---

## 5. Integrations & MoR Strategy

### Payment / Subscription Providers

| Provider | Role | Integration method |
|---|---|---|
| RevenueCat | Primary MoR for native iOS/Android IAP | SDK + Webhook |
| Paddle | Web subscriptions + MoR for web-to-app | Checkout overlay + Webhook |
| Stripe | Fallback web billing; one-time payments | API + Webhook |
| Adapty | Potential alternative to RevenueCat for smaller apps | SDK |

**MoR strategy:** Start with RevenueCat (lowest friction for iOS/Android); expand Paddle for web-to-app flows to own more of the checkout UX. Evaluate own MoR license once platform GMV justifies it.

### Ad Networks

| Network | Integration | Bidding API |
|---|---|---|
| Meta | Marketing API v19 | CPA / value optimization |
| TikTok | Marketing API | CPA |
| Google | Google Ads API | tCPA / tROAS |
| Apple Search Ads | Campaign Management API | CPA |
| AppLovin | MAX SDK + Reporting API | CPE |

### MMP / Attribution

- AppsFlyer, Adjust, Singular — read-only pull via API keys
- S2S postback endpoints registered for key events: `install`, `trial_start`, `subscription`, `renewal`

### CRM / Lifecycle

- Customer.io (primary) — event-triggered sequences
- Braze — alternative for larger clients
- Intercom — in-app messaging + support

### Web Frameworks

- Webflow, Contentful, Next.js — landing page deployment via API for content engine
- Cloudflare Workers — A/B test variant routing at edge

---

## 6. Phased Roadmap

### Phase 0 — Foundation (Weeks 1–6)
**Goal:** Event pipeline, identity graph, basic SDK, payment integration

- [ ] FloKit SDK v1 (iOS + Android): event tracking, identity, anonymous/login merge
- [ ] Ingestion pipeline: Kafka → ClickHouse + PostgreSQL
- [ ] RevenueCat webhook integration + subscription event normalization
- [ ] Funnel analytics dashboard (install → trial → subscription)
- [ ] Basic A/B framework: experiment assignment SDK + results backend
- [ ] MMP integration: read-only AppsFlyer / Adjust pull

### Phase 1 — Acquisition & Creative (Weeks 7–14)
**Goal:** First closed-loop UA optimization + AI creative generation

- [ ] Meta + TikTok bidding agent (daily CPA recalculation + pause rules)
- [ ] Creative asset catalog with performance scoring
- [ ] AI creative brief generator (LLM hooks + platform sizing)
- [ ] ASO keyword tracker + metadata generator
- [ ] Budget allocation engine (nightly, approval-gated)
- [ ] Autopilot Hub v1: hypothesis pool, pending/active/completed view

### Phase 2 — Monetization & Paywall (Weeks 15–22)
**Goal:** Dynamic paywall + offer personalization + MCP offers layer

- [ ] Paywall config service + A/B testing
- [ ] Paddle integration (web-to-app checkout)
- [ ] Dynamic pricing (geo-pricing + segment-based)
- [ ] MCP offer server (get_user_offer, record_interaction endpoints)
- [ ] Paywall benchmark analysis (screenshot upload → variant generation)
- [ ] Revenue cohort analysis dashboard

### Phase 3 — Lifecycle & Retention (Weeks 23–30)
**Goal:** Churn prediction + lifecycle automation + virality

- [ ] Churn prediction model v1 (XGBoost, weekly retrain)
- [ ] Lifecycle state machine + CRM integration (Customer.io)
- [ ] Push notification service (send-time optimization + deep-link routing)
- [ ] Referral system: code gen, deep-link attribution, reward engine
- [ ] Habit formation SDK: streaks, goals, progress events
- [ ] Win-back campaign automation

### Phase 4 — Intelligence & Autopilot (Weeks 31–42)
**Goal:** Full agentic loop + LTV/NBA models + creator tracking

- [ ] LTV prediction model (LightGBM, weekly retrain)
- [ ] Next Best Action bandit (daily update + action dispatch)
- [ ] Creative-to-LTV prediction (CLIP embeddings)
- [ ] Conversion propensity model + early-rescue sequence
- [ ] Content engine (SEO pages, LLM generation, CMS publish API)
- [ ] Creator tracking (UTM + deep-link + cohort attribution)
- [ ] Autopilot agents: Benchmark, Funnel Anomaly, Creative Refresh, Bid Optimization
- [ ] Fraud & abuse detection (rules + ML hybrid)

---

## 7. Target Customer Profile

**Primary ICP:** B2C subscription apps with >$100K MRR, seeking to scale UA efficiency and reduce churn without proportionally scaling the growth team.

**Reference companies:**
- Duolingo (language learning)
- Lightricks / Facetune (photo/video editing)
- Pixelcut (AI image editing)
- LuraHealth (femtech)
- Mid-market fitness, meditation, habit-tracking, gaming apps

**Onboarding flow (for new client apps):**
1. Connect data sources: App Store Connect, MMP, payment provider, ad networks, website analytics
2. FloKit ingests historical data (30–90 days) to bootstrap model training and benchmarks
3. Autopilot Hub populated with initial hypothesis pool based on benchmark gap analysis
4. SDK integrated (2–4 engineer-days); event schema validated
5. First A/B tests launched within Week 1 of integration

**Revenue model:**
- SaaS platform fee (tiered by MAU or MRR managed)
- Rev-share component on attributable revenue lift (optional, for performance-aligned deals)
- Managed service tier: FloKit growth team operates the platform on client's behalf
