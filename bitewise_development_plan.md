# Bitewise: Detailed Development Plan

## Standards To Follow

Bitewise must follow the repository standards in:

- `DEVELOPMENT_GUIDELINES.md`
- `CICD_GUIDELINES.md`

Practical implications:

- TypeScript everywhere.
- Node.js 22.x pinned in `.nvmrc`.
- npm only, with lockfiles committed.
- ESM only: `"type": "module"`.
- `strict: true` TypeScript.
- React Native is approved for `mobile/`.
- React 19 with Vite for `web/`.
- No Tailwind unless the guidelines are amended.
- All services expose `GET /healthz` where relevant.
- Structured JSON logging.
- GitHub Actions from day one.
- `dev` deploys to sandbox.
- `master` deploys to production.

## Repository

Repo:

```text
/Users/shai/git/bitewise
```

GitHub:

```text
git@github.com:shaiboujuP/bitewise.git
```

Folders:

```text
bitewise/
  mobile/
  web/
  api/
  billing/
```

## App Code

Recommended app identifiers:

- Internal app code: `bitewise`
- iOS bundle identifier: `com.bitewise.app`
- Android application ID: `com.bitewise.app`
- App Store Connect SKU: `bitewise-ios`
- Stripe product code: `bitewise-premium-monthly`

## Phase 0: Foundation

Goal:

Make the repo buildable, testable, and aligned with org standards before product features begin.

Tasks:

- Add root `.nvmrc` with Node 22.
- Add root `package.json`.
- Use npm workspaces.
- Add root `package-lock.json`.
- Add shared TypeScript base config.
- Add `.gitignore`.
- Add `.env.example`.
- Add CI workflow.
- Add basic scripts:
  - `npm run typecheck`
  - `npm test`
  - `npm run test:coverage`
  - `npm audit --audit-level=high`

Acceptance criteria:

- `npm ci` works from repo root.
- CI runs on pull requests.
- No folder depends on a package manager other than npm.

## Phase 1: First Mocked Vertical Slice

Goal:

Prove the full V1 user journey before wiring third-party systems.

Journey:

1. Mobile user signs in with mocked auth.
2. User completes onboarding.
3. User receives a calorie target.
4. User starts mocked subscription/trial.
5. User adds mocked meal estimate.
6. User confirms meal.
7. Today view updates.

Work by folder:

- `mobile/`: onboarding, Today view, Add Meal UI.
- `web/`: landing page and basic checkout placeholder.
- `api/`: mock profile, plan, meal, Today endpoints.
- `billing/`: mock entitlement state.

Acceptance criteria:

- Mobile app can complete V1 journey without real external services.
- API returns deterministic mocked data.
- Today view has no dashboard charts.

## Phase 2: Real Auth

Goal:

Replace mocked auth with Google and Apple ID.

Tasks:

- Configure Google OAuth clients.
- Configure Apple Sign In.
- Add auth token validation in `api/`.
- Create or update user on first login.
- Store auth provider IDs.

Acceptance criteria:

- Google login works.
- Apple ID login works.
- API rejects unauthenticated requests.
- Same user identity works across mobile and web.

## Phase 3: Billing With Stripe

Goal:

Enable the 14-day trial and $10/month subscription on web.

Tasks:

- Create Stripe product and monthly price.
- Create checkout session endpoint.
- Create customer portal endpoint.
- Add webhook handler.
- Verify Stripe webhook signatures.
- Map Stripe state to app subscription state.
- Gate meal logging by subscription entitlement.

Subscription states:

```text
trialing
active
past_due
canceled
expired
unknown
```

Acceptance criteria:

- Trial starts in Stripe.
- Webhook updates backend entitlement state.
- Expired/canceled users cannot create new meals.

Important iOS note:

- Stripe web billing is the confirmed web path.
- For App Store approval, Bitewise may still need Apple In-App Purchase for in-app subscription purchase unless the app qualifies for an external purchase entitlement.
- Do not put an in-app link to Stripe checkout into the iOS app until this is resolved.

## Phase 4: API and Database

Goal:

Implement persistent V1 business logic.

Tables:

- `users`
- `profiles`
- `daily_plans`
- `subscriptions`
- `meals`
- `meal_items`
- `food_photos`
- `notification_preferences`
- `analytics_events`

Endpoints:

```text
GET /healthz
GET /api/me
POST /api/profile
POST /api/daily-plan
GET /api/today
POST /api/photos/upload-url
POST /api/meal-estimates
POST /api/meals
PATCH /api/meals/:id
DELETE /api/meals/:id
```

Acceptance criteria:

- All inputs validated at API boundary.
- API returns structured errors.
- Today endpoint returns only V1 Today view data.
- Meal records remain after photo deletion.

## Phase 5: Photo Upload and AI Estimation

Goal:

Replace mocked meal estimates with real photo-based AI estimates.

Tasks:

- Add S3 upload URL endpoint.
- Compress image before upload in mobile.
- Store food photo processing status.
- Call vision-capable AI model.
- Validate structured JSON output.
- Store estimate draft.
- Let user confirm, edit, or retake.
- Schedule 30-day deletion.

Acceptance criteria:

- Valid food photo returns editable estimate.
- Failed estimate shows friendly recovery state.
- AI output is not stored unless schema validation passes.
- Processed photos are deleted after 30 days.

## Phase 6: Minimal Notifications

Goal:

Add only V1 notifications.

Notifications:

- Optional meal reminder.
- Close-to-limit message after meal confirmation.
- Optional end-of-day summary.

Acceptance criteria:

- User can opt out.
- Quiet hours are respected.
- No trial campaign in V1.

## Phase 7: AWS Sandbox and Production

Goal:

Deploy backend services through CI/CD.

AWS services:

- EKS or ECS for deployable services.
- ECR for Docker images.
- RDS Postgres.
- S3 for photos.
- Secrets Manager.
- CloudWatch.
- EventBridge Scheduler.

Guideline alignment:

- Sandbox deploys from `dev`.
- Production deploys from `master`.
- GitHub Actions builds Docker images.
- Smoke tests call `/healthz`.

Acceptance criteria:

- Sandbox URL is live.
- Production URL is live.
- CI deploys sandbox automatically from `dev`.
- Production deploys only after PR merge into `master`.

## Phase 8: Beta Release

Goal:

Release Bitewise to a small beta group.

Tasks:

- Prepare TestFlight.
- Prepare Android internal testing if Android is included in first beta.
- Validate auth.
- Validate Stripe trial.
- Validate entitlement gating.
- Validate AI estimates.
- Validate photo deletion job.
- Validate privacy and terms links.

Acceptance criteria:

- Beta users can complete the V1 flow.
- No blocker errors in logs.
- Billing and entitlement state are reliable.

## Agent Work Split

Agent 1: Repo Foundation

- Root npm workspace.
- Node 22.
- TypeScript strict config.
- CI skeleton.
- GitHub branch flow.

Agent 2: Mobile

- Expo React Native.
- Auth screens.
- Onboarding.
- Today view.
- Add Meal flow.
- Estimate review.

Agent 3: Web

- React 19 with Vite.
- Landing page.
- Auth entry.
- Stripe checkout/account links.

Agent 4: API

- Node service.
- Health endpoint.
- Auth middleware.
- Database schema.
- Profile, plan, meal, Today endpoints.

Agent 5: Billing

- Stripe product setup.
- Checkout.
- Portal.
- Webhooks.
- Entitlement sync.

Agent 6: AI and Storage

- S3 upload.
- AI estimate.
- Schema validation.
- Photo retention deletion.

Agent 7: AWS and CI/CD

- ECR.
- Runtime deployment.
- GitHub Actions.
- Sandbox and production.
- Smoke tests.

Agent 8: QA and Release

- Test plan.
- App Store readiness.
- Billing tests.
- Auth tests.
- AI estimate tests.

