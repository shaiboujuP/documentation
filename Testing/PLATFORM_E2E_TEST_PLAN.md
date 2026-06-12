# FloKit Platform — End-to-End Test Plan

Single source of truth for platform-wide E2E testing. Validates **all FloKit features** against the
live platform using dedicated test accounts. Catches "deployed but not wired" failures (dead
workers, bad keys, broken service-to-service calls) that per-service unit tests cannot. Tests assert
on **feature outcomes and invariants**, never on nondeterministic output (GPT scores, generated copy).

---

## 1. System under test

| Service | Host | Role | Auth |
|---|---|---|---|
| website | flokitai.com | Marketing site + contact form + PMF pages | none |
| dashboard | dashboard.flokitai.com / api.dashboard.flokitai.com | Internal admin: campaigns, overview, users | login/session |
| bitewise-api | bitewise-api.flokitai.com | **Test/demo app** (calorie tracker) — used as a test harness, not a FloKit product feature | Google/Apple → JWT |
| campaign-manager | campaign-manager.flokitai.com | A/B campaigns + variants (feeds dashboard) | service |
| payments-gateway | payments-gateway.flokitai.com | Public payments entrypoint | service |
| payments-api | payments-api.flokitai.com | Billing + Stripe webhooks (gateway→api→mongo→stripe) | service |
| web2app-manager | (internal) | App Store URL → generated landing page | none |
| agentic-pmf-test | agentic-pmf-test.flokitai.com | PMF scoring (queue+worker+GPT) | magic-link → JWT |
| devops | — | k3s/EC2, NLB, nginx-ingress, Terraform (platform) | — |

## 2. Testing infrastructure

- **Runner:** **Vitest + `fetch`** for API-level feature tests (already the standard suite across
  every service) and **Playwright** for browser journeys (website signup, dashboard login,
  bitewise web).
- **Home:** a **dedicated new repo — `platform-e2e`** (not inside any one service), so the suite is
  owned independently and can target every host. See §3.
- **Test accounts & isolation:** dedicated per feature — `e2e+pmf@flokit.ai` (PMF magic-link), a
  dashboard test admin, seeded test campaigns (`testRun`-tagged). Bitewise is reserved as the
  payments test harness for the next wave.
- **Test hooks:** a guarded `POST /__test__/reset` + token/OTP fetch per service, enabled only when
  `E2E_TEST_HOOKS=1` and authorized by an `X-E2E-Secret` header (extends bitewise's existing
  `testReset` helper to a prod-safe endpoint). The endpoints 404 when the flag is off.
- **Secrets:** pulled from **AWS Secrets Manager** (same paths the deploys use) — test creds, JWT
  secrets, Stripe test keys, `E2E_SECRET`. Injected per CI environment, never committed.
- **Orchestration & reporting:** GitHub Actions in `platform-e2e`; results to the existing **Slack**
  webhook; scheduled-monitor failures escalate to PagerDuty.

## 3. Repo & layout

**`platform-e2e`** — a new standalone repository.

```
platform-e2e/
  package.json              # vitest + @playwright/test
  src/
    client.ts               # fetch wrapper, per-host base URLs, auth helpers, pollUntil()
    hooks.ts                # guarded /__test__ reset + token/OTP fetch
    fixtures.ts             # test accounts, Stripe test ids, stable PMF fixture URL
    api/   *.e2e.ts          # API-level feature journeys (vitest)
    ui/    *.spec.ts         # browser journeys (playwright)
  vitest.config.ts
  playwright.config.ts
  .github/workflows/prod-e2e.yml
  docs/CREATE_TEST_ACCOUNTS.md   # account-provisioning brief (run once)
```

Per-service repos change only minimally: each deploy job emits a `repository_dispatch` to
`platform-e2e` after a successful rollout, and each service exposes the guarded `/__test__` hook.

## 4. How & when it runs

| Trigger | Scope | Environment | Action on fail |
|---|---|---|---|
| **Post-deploy gate** — `repository_dispatch` from a service's deploy job after rollout | That service's feature journeys | sandbox first, then prod | Mark deploy unhealthy; Slack alert |
| **Scheduled monitor** — cron every 15 min | Critical revenue/auth journeys | prod | Slack; page on-call after 2 consecutive fails |
| **Nightly full** — cron daily 02:00 UTC | Entire suite, all services | prod (Stripe test mode) | Slack digest; open issue |
| **On-demand** — `workflow_dispatch` (env + tag filter) | Operator's choice | chosen | Report only |

The post-deploy gate replaces the bare `curl /healthz` smoke step each service runs today.

## 5. Main rules for running

1. **Test accounts only.** Every request is scoped to a test tenant; never reads or writes real user data.
2. **Stripe test mode only.** Enforced by a startup assertion that the key is `sk_test_*`; live keys are never present.
3. **Writes go through guarded hooks.** Prod mutations only via `/__test__` endpoints (`E2E_TEST_HOOKS=1` + `X-E2E-Secret`).
4. **Run-isolated & idempotent.** Each run namespaces data by `runId`; any test runs alone and repeatably.
5. **Non-destructive.** Deletes/resets scoped by test-tenant id; a guard refuses any id without the `e2e-` prefix.
6. **Fail-fast smoke, isolated journeys.** Independent service journeys use `continue-on-error` so one broken service doesn't mask the rest.
7. **Cost-capped.** ≤2 GPT scoring runs and a bounded number of Firecrawl/scrape calls per run.
8. **Retries bounded.** Max 2 retries with backoff; persistently flaky tests are quarantined (`@flaky`, nightly-only).
9. **Secrets from AWS Secrets Manager only.**
10. **Every run reports** to Slack; scheduled-monitor failures escalate to PagerDuty.

## 6. What we do with the data

- **Synthetic only, namespaced.** Records carry `{ testRun: <runId> }`; addresses like `e2e+<runId>@flokit.ai`. No PII, no production data touched.
- **Created via hooks, torn down in `afterAll`.** Each spec calls `/__test__/reset` scoped to its test tenant; the run's final step sweeps anything left by `runId`.
- **Stripe:** test-mode customers/subscriptions created per run and cancelled at teardown.
- **Mongo backstop:** a TTL index on `testRun` collections expires orphans (24h) if a run crashes before cleanup.
- **CI artifacts:** Playwright traces/screenshots and HTTP logs kept 7 days, then auto-deleted.

## 7. On which environment

- **Sandbox/dev first where it exists** (e.g. agentic-pmf-test's `dev`-branch sandbox) — the post-deploy gate runs full journeys here before promotion.
- **Production** is the primary target for synthetic monitoring and post-promote smoke — the "test account in prod" requirement. Writes confined to test tenants via the hooks above.
- **Stripe is always test mode**, in every environment, enforced by the `sk_test_*` assertion.

---

## 8. The first 10 E2E tests — 10 main product features

Each test exercises one user-facing feature end-to-end.
Format: **Feature · User goal · Preconditions · Steps · Expected result · Test data / cleanup.**

### F1 — PMF: passwordless magic-link sign-in
*User goal:* a founder signs in with just an email, no password.
*Pre:* PMF test hook on. *Steps:* `POST /api/auth/magic-link` {email, companyName} → fetch the OTP via
`/__test__/latest-otp` → `GET /api/auth/verify?token=`. *Expected:* `{sent:true}`; verify returns a
valid JWT and a user record with the submitted email/company; the OTP is single-use (replay → 400).
*Data/cleanup:* reset PMF tokens for the test email.

### F2 — PMF: product PMF-readiness scoring report
*User goal:* submit a product URL and receive a scored readiness report.
*Pre:* PMF JWT; stable fixture product URL. *Steps:* `POST /api/score` {productUrl, inputMethod:'url'}
→ poll `GET /api/score/:id` (≤90s). *Expected:* 202 + reportId; status `pending`→`processing`→
`complete`; report has 5 pillars each 0–20, `totalScore == Σpillars` (0–100), `readiness` matches the
≥65/≥40 threshold rule, non-empty `verdict` + `recommendedAngle`. *(Requires the PMF worker to be
deployed — see §9.)* *Data/cleanup:* delete the report by id.

### F3 — PMF: report history ("my reports")
*User goal:* a returning user sees their past scoring reports and only their own.
*Pre:* two PMF JWTs (user A, user B); ≥2 reports owned by A. *Steps:* `GET /api/score` as A; `GET
/api/score/:id` as B for one of A's reports. *Expected:* A's list returns exactly A's reports
(newest-first, summary fields present); cross-user fetch → 403; unknown id → 404. *Data/cleanup:*
delete the seeded reports.

### F4 — web2app: generate a landing page from an App Store URL
*User goal:* paste an App Store URL and get a generated landing page.
*Pre:* web2app reachable; a stable public App Store fixture URL. *Steps:* `POST /api/generate`
{appStoreUrl}. *Expected:* 200; `appId` extracted; response carries `AppStoreData` populated from
Apple lookup (name, developer, description, ≥1 screenshot) and the generated page sections. *Data/cleanup:* none.

### F5 — web2app: graceful handling of an invalid App Store URL
*User goal:* a bad/non-existent URL produces a clear error, not a crash.
*Pre:* web2app reachable. *Steps:* `POST /api/generate` with (a) a non-App-Store URL and (b) a
well-formed App Store URL for a non-existent app id. *Steps:* also a non-POST/`/api/generate` request.
*Expected:* 4xx with a structured `{error}` body; service stays healthy (no 5xx, `/healthz` still 200).
*Data/cleanup:* none.

### F6 — Campaign Manager: autonomous variant evaluation & decisioning
*User goal:* the optimizer scores live ad variants and decides scale/hold/kill automatically.
*Pre:* campaign hook on; seed one campaign with variants carrying known KPIs (a strong ROAS variant
and a poor one). *Steps:* trigger one scheduler tick via `/__test__/tick` → `GET /api/campaigns/:id`.
*Expected:* each variant has a fresh evaluation with a `score` and a `decision`; the strong variant's
decision favours scaling, the poor one trends toward kill (`consecutivePoorRuns` increments).
*Data/cleanup:* delete the seeded campaign by `testRun`.

### F7 — Campaign Manager: evolutionary variant generation
*User goal:* winning variants spawn a new generation; chronic losers are retired.
*Pre:* seeded campaign with a top-performing variant and a variant past the poor-run threshold.
*Steps:* run enough `/__test__/tick` cycles to trigger evolution → `GET /api/campaigns/:id/variants`.
*Expected:* a new variant appears at `generation+1` with `parentVariantId` = the winner; the chronic
loser moves to a non-active `status`. *Data/cleanup:* delete the seeded campaign by `testRun`.

### F8 — Dashboard: admin sign-in & overview
*User goal:* an operator logs into the control plane and sees platform metrics.
*Pre:* dashboard test-admin creds. *Steps:* `POST /api/auth/login` → `GET /api/auth/me` → `GET
/api/overview`; then `GET /api/overview` with no session. *Expected:* login establishes a valid
session; `/me` returns the test admin; `/overview` returns well-formed metrics; unauthenticated → 401.
*Data/cleanup:* `POST /api/auth/logout`.

### F9 — Dashboard: campaign data propagation (cross-service)
*User goal:* campaigns optimized by Campaign Manager are visible in the dashboard.
*Pre:* dashboard admin session; the seeded campaign from F6 present. *Steps:* `GET dashboard
api/campaigns` and `api/campaigns/:id/variants`. *Expected:* the campaign + its variants surface
through the dashboard API with KPI fields (score, decision, ROAS) matching what Campaign Manager
exposes — proving the FE→Dashboard API→Campaign Manager→Mongo chain. *Data/cleanup:* via F6 teardown.

### F10 — Website: PMF signup funnel (browser)
*User goal:* a visitor on the marketing site requests access through the PMF signup modal.
*Pre:* Playwright; PMF test hook on. *Steps:* load the PMF marketing page → open `SignupModal` →
submit test email + company → submit. *Expected:* the UI confirms "magic link sent"; a magic-link
token exists for the test email (verify via `/__test__/latest-otp`); the call hit the real PMF API.
*Data/cleanup:* reset PMF tokens for the test email.

### Coverage map
PMF F1–F3 · web2app F4–F5 · Campaign Manager F6–F7 · Dashboard F8–F9 · Website F10.
**Payments is intentionally out of this first wave** (de-prioritised). It will be covered separately
using **Bitewise as the test consumer/harness** to exercise the gateway→api→Stripe-test-mode→webhook
chain. Other next-wave items: dashboard user management, website contact form, web2app page-render UI.

---

## 9. Prerequisites

1. **Deploy the PMF pipeline worker** — its `Dockerfile` only starts the API server, so jobs enqueue
   but never run and reports stay `pending`. *(Real bug; F2 fails until fixed.)*
2. Add the guarded `/__test__` hooks per service: PMF (`reset`, `latest-otp`), Campaign Manager
   (`reset`, `tick` to advance the scheduler deterministically), web2app (none needed — stateless),
   all behind `E2E_TEST_HOOKS` + `E2E_SECRET`.
3. Create the test accounts (see `platform-e2e/docs/CREATE_TEST_ACCOUNTS.md`) and inject host URLs +
   secrets into the CI environment from AWS Secrets Manager.
4. *(Payments next wave only)* Stripe **test mode** keys + test customer, with Bitewise as the
   consumer/harness.

---

## 10. Technical architecture

**Stack:** Node 22 · TypeScript (ESM, run via `tsx`) · **Vitest** for API-level features · **Playwright**
for the one browser feature (F10). No web framework — just `fetch` + the runners.

### 10.1 Directory layout

```
platform-e2e/
  src/
    lib/                       ← built in Phase 1, then FROZEN. Phases 2–4 import only.
      config.ts                env + target resolution, base URLs, runId, env guards
      http.ts                  HttpClient (fetch wrapper, retry, pollUntil)
      hooks.ts                 typed callers for /__test__/* (reset, latestOtp, seedCampaign, tick)
      auth.ts                  token helpers (pmfMagicLinkSignIn, mintPmfJwt, dashboardLogin)
      report.ts                result collector → results.json (+ Slack in Phase 5)
      types.ts                 response shapes mirrored from the services
    tests/                     ← one SELF-CONTAINED file per feature
      f01-pmf-signin.e2e.ts    f02-pmf-score.e2e.ts    f03-pmf-history.e2e.ts
      f04-web2app-generate.e2e.ts   f05-web2app-invalid.e2e.ts
      f06-campaign-eval.e2e.ts      f07-campaign-evolve.e2e.ts
      f08-dashboard-overview.e2e.ts f09-dashboard-propagation.e2e.ts
      f10-website-signup.spec.ts    (Playwright)
  vitest.config.ts   playwright.config.ts
  .github/workflows/prod-e2e.yml
  docs/CREATE_TEST_ACCOUNTS.md
```

### 10.2 Parallelization contract (the reason this splits cleanly)

- **Phase 1 ships and freezes `src/lib/`** plus all 10 test files as **passing skeletons**.
- **Phases 2, 3, 4 each create/edit ONLY their own `tests/fNN-*` files.** They never modify `src/lib/`
  or another phase's test file.
- Every test file is **self-contained**: it imports from `src/lib/` and declares its own fixtures
  inline (no shared `fixtures.ts`).
- Result: the three phases touch **disjoint file sets** → no merge conflicts → run in parallel.
- If a phase thinks it needs a new `lib` capability, that's a Phase-1 gap to fix first — not an edit
  to a frozen file mid-parallel-work.

### 10.3 Module contracts (signatures Phase 1 must deliver)

- `config.ts` → `getConfig()`: `{ env: 'sandbox'|'prod', baseUrls: Record<Service,string>, e2eSecret,
  runId }`. `runId = e2e-<crypto.randomUUID slice>` minted once per process. Throws if required env
  missing; for payments (next wave) asserts the Stripe key is `sk_test_*`.
- `http.ts` → `class HttpClient(baseUrl)` with `get/post/put/del(path, {json?, headers?, bearer?})`
  → `{status, body}`; `pollUntil<T>(fn, predicate, {timeoutMs, intervalMs})`; idempotent GET retry
  ≤2 with backoff.
- `hooks.ts` → `reset(service, tenantId)`, `latestOtp(email)`, `seedCampaign(spec)`, `tick(id)` —
  all inject the `X-E2E-Secret` header; throw a clear error if hooks are disabled.
- `auth.ts` → `pmfMagicLinkSignIn(email, company)`→`{jwt,user}`, `mintPmfJwt(userId,email)`,
  `dashboardLogin(creds)`→`session`.
- `report.ts` → a Vitest reporter that writes `results.json` (`{feature,status,durationMs,error}[]`);
  Phase 5 adds the Slack formatter.

### 10.4 Env contract

Injected by CI from `flokit-prod/e2e/*`: `TARGET_ENV`, `E2E_SECRET`, per-host `*_BASE_URL`,
`PMF_JWT_SECRET`, `DASHBOARD_ADMIN_USER`/`PASS`, fixture URLs (`PMF_FIXTURE_URL`,
`WEB2APP_FIXTURE_URL`). **Skeletons pass with no env** (they don't hit the network). Real
implementations gate on env via `describe.skipIf(!hasEnv)` so local dev isn't blocked, while CI —
where env is always present — runs them for real.

### 10.5 CI workflow & selective runs

`prod-e2e.yml` triggers exactly as §4 (repository_dispatch post-deploy · cron monitor · nightly ·
workflow_dispatch). Jobs: **setup** (resolve env/secrets) → **matrix** sharded by group
(`pmf`=f01–03, `mid`=f04–06, `tail`=f07–10, `ui`=f10 Playwright) each running a file glob →
**report** (aggregate `results.json`, post to Slack). A `@critical` test tag selects the 15-minute
monitor subset.

---

## 11. Phased delivery plan (plan only — do not execute yet)

Five phases. **Phase 1 must complete and merge first.** Then **Phases 2, 3, 4 run fully in parallel**
(disjoint files). **Phase 5 runs last.** Each phase below carries a self-contained prompt to hand to
another agent.

| Phase | Scope | Depends on | Files it owns |
|---|---|---|---|
| 1 | Repo + mechanism + 10 passing skeletons + CI + reporting stub | — | `src/lib/*`, `tests/f01–f10` (skeletons), configs, workflow |
| 2 | Implement F1–F3 (PMF) | 1 | `tests/f01,f02,f03` |
| 3 | Implement F4–F6 (web2app ×2, campaign ×1) | 1 | `tests/f04,f05,f06` |
| 4 | Implement F7–F10 (campaign ×1, dashboard ×2, website UI) | 1 | `tests/f07,f08,f09,f10` |
| 5 | Whole-system validation + Slack reporting + status report | 2,3,4 | `src/lib/report.ts`, workflow `report` job, `docs/REPORT_FORMAT.md` |

### Phase 1 — Repo & mechanism (prompt)
```
Build the `platform-e2e` repo skeleton and the shared test mechanism. The repo already exists at
/Users/shai/git/flokit.ai/platform-e2e with README, package.json, .gitignore, and
docs/CREATE_TEST_ACCOUNTS.md — extend it; do not rewrite those.

Deliver:
1. src/lib/ with these modules and the EXACT signatures in §10.3 of
   documentation/Testing/PLATFORM_E2E_TEST_PLAN.md: config.ts, http.ts (HttpClient + pollUntil),
   hooks.ts, auth.ts, report.ts (Vitest reporter → results.json), types.ts.
2. vitest.config.ts (uses the report.ts reporter; testTimeout 120s) and playwright.config.ts.
3. The 10 test files in src/tests/ named exactly as §10.1, each a PASSING SKELETON: real describe/it
   structure, runId + cleanup hooks wired, but assertions are placeholders (`expect(true).toBe(true)`
   with a `// TODO Fxx:` note). They MUST pass with no env set and hit no network.
4. .github/workflows/prod-e2e.yml per §10.5: triggers (repository_dispatch, schedule, workflow_dispatch),
   a matrix sharded pmf/mid/tail/ui by file glob, and a report job that runs report.ts and (stub) Slack.
5. A one-line `npm run test:e2e` that runs green locally and in CI.

HARD RULES: src/lib/ is the FROZEN contract for phases 2–4 — design the signatures so F1–F10 can be
implemented by editing ONLY their own test file. Each test file is self-contained (imports lib,
declares fixtures inline). Do not implement real feature assertions — that is phases 2–4.
Acceptance: `npm run test:e2e` exits 0 with 10 passing skeletons; CI workflow is valid and green.
```

### Phase 2 — F1–F3, PMF (prompt)
```
Implement the real assertions for F1, F2, F3 in platform-e2e. Edit ONLY src/tests/f01-pmf-signin.e2e.ts,
f02-pmf-score.e2e.ts, f03-pmf-history.e2e.ts. Import everything shared from src/lib/ (frozen — do not
modify it or any other test file). Gate network tests with describe.skipIf(!hasEnv).

Target service: agentic-pmf-test (PMF_BASE_URL). Endpoints:
- POST /api/auth/magic-link {email, companyName} → {sent:true}
- GET  /api/auth/verify?token=... → {token(JWT), user}
- POST /api/score {productUrl|appStoreUrl|playStoreUrl|description, inputMethod:'url'|...} → 202 {reportId}
- GET  /api/score/:id  ·  GET /api/score (list)
Hooks: GET /__test__/latest-otp?email= ; POST /__test__/reset.

F1 sign-in: magic-link → latestOtp → verify → assert valid JWT + user fields; OTP single-use (replay→400).
F2 scoring: POST /api/score with PMF_FIXTURE_URL → pollUntil status==='complete' (≤90s); assert 5 pillars
   0–20, totalScore==Σ and 0–100, readiness matches ≥65/≥40 rule, verdict+recommendedAngle non-empty.
   (Will fail until the PMF worker is deployed — note that, don't hack around it.)
F3 history: seed ≥2 reports for user A, 1 for user B; GET /api/score as A returns only A's; cross-user
   GET /api/score/:id → 403; unknown id → 404.
Cleanup: afterAll → hooks.reset for the test email/runId. Use e2e+<runId>@flokit.ai addresses.
Acceptance: the 3 files pass in CI; no edits outside them.
```

### Phase 3 — F4–F6, web2app + campaign (prompt)
```
Implement F4, F5, F6 in platform-e2e. Edit ONLY src/tests/f04-web2app-generate.e2e.ts,
f05-web2app-invalid.e2e.ts, f06-campaign-eval.e2e.ts. Import shared code from src/lib/ (frozen).
Gate network tests with describe.skipIf(!hasEnv).

web2app-manager (WEB2APP_BASE_URL), stateless:
- POST /api/generate {appStoreUrl} → 200 AppStoreData {appId,name,developer,description,screenshotUrls[],...}
F4 generate: POST with WEB2APP_FIXTURE_URL (a stable real App Store URL) → assert appId extracted and
   name/developer/description present, ≥1 screenshot.
F5 invalid: POST a non-App-Store URL AND a well-formed App Store URL for a non-existent app id →
   each returns 4xx with {error}; then a non-POST/other path → 4xx; /healthz still 200 afterward.

campaign-manager (CAMPAIGN_BASE_URL), read-only API + guarded hooks:
- GET /api/campaigns/:id  ·  GET /api/campaigns/:id/variants
- Hooks: POST /__test__/seed-campaign (creates campaign + variants w/ given KPIs, testRun-tagged,
  returns id) ; POST /__test__/tick (runs ONE evaluation cycle) ; POST /__test__/reset.
F6 eval: seedCampaign with one strong-ROAS variant and one poor variant → hooks.tick(id) →
   GET /api/campaigns/:id : assert each variant has a fresh evaluation with score+decision; strong
   variant's decision trends to scale, poor variant's consecutivePoorRuns increments.
Cleanup: afterAll → hooks.reset('campaign', runId). Tag all seeded data with the runId.
Acceptance: the 3 files pass in CI; no edits outside them.
```

### Phase 4 — F7–F10, campaign + dashboard + website (prompt)
```
Implement F7, F8, F9, F10 in platform-e2e. Edit ONLY src/tests/f07-campaign-evolve.e2e.ts,
f08-dashboard-overview.e2e.ts, f09-dashboard-propagation.e2e.ts, f10-website-signup.spec.ts (Playwright).
Import shared code from src/lib/ (frozen). Gate network tests with skipIf(!hasEnv).

F7 evolve (campaign-manager): seedCampaign with a top performer and a variant already past the
   poor-run threshold → run enough hooks.tick cycles to trigger evolution → GET /api/campaigns/:id/variants:
   assert a new variant at generation+1 with parentVariantId = the winner; the chronic loser is non-active.
   Cleanup: hooks.reset('campaign', runId).

Dashboard (DASHBOARD_API_BASE_URL): POST /api/auth/login {DASHBOARD_ADMIN_USER/PASS} → session;
   GET /api/auth/me ; GET /api/overview ; GET /api/campaigns ; GET /api/campaigns/:id/variants.
F8 overview: login → /me returns the admin → /overview returns well-formed metrics; no session → 401;
   afterAll POST /api/auth/logout.
F9 propagation: with a seeded campaign present (reuse the F6/F7 seed pattern via hooks), GET dashboard
   /api/campaigns + /api/campaigns/:id/variants and assert the campaign + variants surface with KPI
   fields (score, decision, roas) — proving FE→Dashboard API→Campaign Manager→Mongo.

F10 website signup (Playwright, WEBSITE_BASE_URL): load the PMF marketing page → open SignupModal →
   fill test email (e2e+<runId>@flokit.ai) + company → submit → assert the "magic link sent" UI state;
   then verify a token exists via hooks.latestOtp. Cleanup: hooks.reset('pmf', email).
Acceptance: the 4 files pass in CI; no edits outside them.
```

### Phase 5 — Whole-system validation & reporting (prompt)
```
Validate platform-e2e end-to-end and finish reporting. Now you MAY edit src/lib/report.ts and the
workflow's report job (phases 2–4 are merged). Do not change feature assertions except to fix genuine
harness bugs (and note any you find).

1. Run the full suite (all 10) against the SANDBOX target, then a read-only subset against PROD.
   Triage failures: real product bug vs. harness bug vs. missing prerequisite (e.g. PMF worker, a
   /__test__ hook). Produce a findings list — do not paper over product bugs.
2. Implement report.ts Slack reporting: aggregate results.json into a compact message — overall
   pass/fail, per-feature ✅/❌ with duration, links to CI run + Playwright artifacts; red on any fail.
   Post via the existing Slack webhook (SLACK_WEBHOOK_URL). Keep it to ~15 lines.
3. Wire the workflow `report` job to always run (success or failure) and post that message; escalate
   to PagerDuty only for the scheduled-monitor trigger after 2 consecutive failures.
4. Write docs/REPORT_FORMAT.md: a short spec of the status report (fields, example Slack block,
   how to read a failure).
Acceptance: a single `workflow_dispatch` run executes all 10, writes results.json, and posts one
clear Slack status message; REPORT_FORMAT.md committed.
```
