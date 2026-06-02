# Bitewise: Mobile API Backend Service Agent Brief

## Purpose

This document is for a spawn agent that will build the API/backend service supporting the Bitewise mobile app.

Repo:

```text
/Users/shai/git/bitewise
```

API folder:

```text
/Users/shai/git/bitewise/api
```

The API service must support the V1 mobile loop:

1. Authenticate user.
2. Complete onboarding.
3. Validate entitlement.
4. Create daily calorie plan.
5. Upload food photo.
6. Generate AI meal estimate.
7. Confirm or edit meal.
8. Return Today view.

No full dashboard is required in V1.

## Required Standards

Follow:

- `/Users/shai/git/documentation/DEVELOPMENT_GUIDELINES.md`
- `/Users/shai/git/documentation/CICD_GUIDELINES.md`

Key rules:

- TypeScript only.
- Node.js 22.x.
- npm only.
- ESM only.
- `strict: true`.
- No Express.
- Prefer Node `http.createServer` or Hono.
- Export `createApp()` so tests can run without binding a port.
- All routes under `/api/`, except `GET /healthz`.
- Structured JSON errors: `{ error: string }`.
- Never leak stack traces to clients.
- Validate all input at the API boundary.
- Every outbound request must use a timeout.
- Log structured JSON only.
- Do not log PII, tokens, secrets, or full request bodies.

## Recommended API Structure

```text
api/
  src/
    server/
      createApp.ts
      index.ts
      routes/
        health.ts
        me.ts
        profile.ts
        dailyPlan.ts
        today.ts
        purchases.ts
        photos.ts
        mealEstimates.ts
        meals.ts
      middleware/
        auth.ts
        requestLogging.ts
        errorHandler.ts
        entitlementGuard.ts
    shared/
      types.ts
      schemas.ts
      errors.ts
      constants.ts
    services/
      authService.ts
      profileService.ts
      caloriePlanService.ts
      entitlementService.ts
      photoService.ts
      aiMealEstimateService.ts
      mealService.ts
      todayService.ts
    repositories/
      userRepository.ts
      profileRepository.ts
      subscriptionRepository.ts
      mealRepository.ts
      photoRepository.ts
    tests/
      unit/
      e2e/
```

Use this structure unless the current repo has already established a different local pattern.

## Primary Mobile Contracts

The mobile app needs these endpoints.

```text
GET /healthz
GET /api/me
POST /api/profile
POST /api/daily-plan
GET /api/today
GET /api/purchase-options
POST /api/purchases/iap/verify
POST /api/purchases/web-checkout-session
POST /api/photos/upload-url
POST /api/meal-estimates
GET /api/meal-estimates/:id
POST /api/meals
PATCH /api/meals/:id
DELETE /api/meals/:id
```

V1 should not expose a full analytics dashboard endpoint.

## Authentication

Supported auth providers:

- Google.
- Apple ID.

API responsibility:

- Validate auth token.
- Resolve or create app user.
- Attach `userId` to request context.
- Reject unauthenticated requests with `401`.

Do not store provider tokens unless absolutely required.

Do not use Stripe, Apple purchase ID, or Google purchase ID as the app identity.

## Entitlements and Purchase Options

The API must support two mobile purchase options:

1. In-app purchase.
2. Link out to web checkout.

Entitlement model:

```text
premium
```

Subscription providers:

```text
stripe
apple
google
```

Subscription states:

```text
trialing
active
past_due
canceled
expired
unknown
```

Rules:

- Meal logging requires `trialing` or `active`.
- API is the source of truth for app entitlement decisions.
- Mobile must never unlock premium based only on local purchase state.
- Stripe, Apple, and Google all map into the same internal entitlement model.

### GET /api/purchase-options

Returns purchase paths available to this user/device/region.

Example response:

```json
{
  "options": [
    {
      "id": "iap",
      "available": true,
      "provider": "apple",
      "label": "Start free trial"
    },
    {
      "id": "web_linkout",
      "available": false,
      "provider": "stripe",
      "label": "Subscribe on web",
      "unavailableReason": "Not available in this region"
    }
  ]
}
```

Implementation notes:

- Web link-out must be feature-flagged.
- Web link-out must be region/programme gated.
- Default to disabled unless explicitly allowed.

## Profile and Onboarding

### POST /api/profile

Stores onboarding inputs:

- Age.
- Gender.
- Height.
- Weight.
- Goal.
- Activity level.
- Target pace.
- Optional dietary restrictions.

Validation:

- Reject impossible ages, heights, or weights.
- Reject unknown enum values.
- Return `400` for invalid input.

### POST /api/daily-plan

Creates or updates the active daily calorie plan.

Inputs:

- Profile inputs.
- Optional manual calorie target override.

Output:

- Daily calorie target.
- Calculation metadata.
- Active plan ID.

Implementation rule:

- Keep calorie target calculation deterministic.
- Do not call AI for V1 calorie target calculation.

## Today View

### GET /api/today

Returns the mobile home surface.

Example response:

```json
{
  "date": "2026-06-02",
  "timezone": "Asia/Jerusalem",
  "calorieTarget": 2000,
  "caloriesConsumed": 620,
  "caloriesRemaining": 1380,
  "status": "on_track",
  "meals": [
    {
      "id": "meal_123",
      "name": "Chicken salad with rice",
      "calories": 620,
      "createdAt": "2026-06-02T11:40:00.000Z"
    }
  ]
}
```

Status values:

```text
on_track
close_to_limit
over_target
```

Status calculation:

```text
remaining = calorieTarget - caloriesConsumed

if remaining > 300: on_track
if remaining >= 0 and remaining <= 300: close_to_limit
if remaining < 0: over_target
```

No dashboard charts in V1.

## Photo Upload

### POST /api/photos/upload-url

Creates a signed upload URL for a food photo.

Inputs:

- File name.
- Content type.
- Approximate file size.

Output:

- `photoId`.
- Signed upload URL.
- Required upload headers.
- Expiration timestamp.

Rules:

- Only allow image MIME types.
- Enforce maximum upload size.
- Store photo status as `upload_pending`.
- Store user ownership.

Photo retention:

- Delete processed food photos 30 days after processing.
- Meal records remain after photo deletion.

Recommended states:

```text
upload_pending
uploaded
processing
processed
failed
deleted
```

## AI Meal Estimation

### POST /api/meal-estimates

Creates an estimate from an uploaded photo.

Inputs:

- `photoId`.

Process:

1. Confirm photo belongs to user.
2. Confirm user has active entitlement.
3. Mark photo as `processing`.
4. Call vision-capable AI model.
5. Validate structured output.
6. Store estimate draft.
7. Mark photo as `processed` or `failed`.

Example output:

```json
{
  "id": "estimate_123",
  "mealName": "Chicken salad with rice",
  "estimatedCalories": 620,
  "confidence": "medium",
  "items": [
    {
      "name": "chicken breast",
      "estimatedCalories": 240,
      "proteinGrams": 38,
      "carbsGrams": 0,
      "fatGrams": 8
    }
  ],
  "macros": {
    "proteinGrams": 38,
    "carbsGrams": 55,
    "fatGrams": 22
  }
}
```

Validation:

- Calories must be positive.
- Confidence must be `low`, `medium`, or `high`.
- Bad AI output must not be saved as confirmed meal.
- Low confidence is allowed, but must be visible to mobile.

## Meal Confirmation

### POST /api/meals

Creates a confirmed meal from an estimate or manual edit.

Inputs:

- Estimate ID, optional.
- Photo ID, optional.
- Meal name.
- Calories.
- Optional macros.
- Meal items.

Rules:

- User must have active entitlement.
- Calories must be positive.
- Confirmed meal updates Today view immediately.

### PATCH /api/meals/:id

Allows editing:

- Meal name.
- Calories.
- Optional macros.
- Meal items.

### DELETE /api/meals/:id

Soft delete is preferred.

Deleting a meal updates Today view.

## Database Tables

Minimum V1 tables:

```text
users
profiles
daily_plans
subscriptions
food_photos
meal_estimates
meals
meal_items
analytics_events
```

Important columns:

`users`

- `id`
- `auth_provider`
- `auth_provider_subject`
- `created_at`
- `updated_at`

`subscriptions`

- `id`
- `user_id`
- `provider`
- `provider_customer_id`
- `provider_subscription_id`
- `status`
- `entitlement`
- `current_period_end`
- `created_at`
- `updated_at`

`food_photos`

- `id`
- `user_id`
- `storage_key`
- `status`
- `processed_at`
- `delete_after`
- `deleted_at`

`meals`

- `id`
- `user_id`
- `date`
- `timezone`
- `name`
- `calories`
- `protein_grams`
- `carbs_grams`
- `fat_grams`
- `deleted_at`

## Error Model

Return:

```json
{
  "error": "Human readable message"
}
```

Recommended known error codes internally:

```text
UNAUTHENTICATED
FORBIDDEN
VALIDATION_ERROR
ENTITLEMENT_REQUIRED
PHOTO_NOT_FOUND
PHOTO_NOT_READY
ESTIMATE_FAILED
MEAL_NOT_FOUND
INTERNAL_ERROR
```

Do not expose stack traces.

## Logging and Observability

Log structured JSON:

- Request received.
- Request completed.
- Validation errors.
- Entitlement failures.
- Photo upload URL created.
- AI estimate started.
- AI estimate completed.
- AI estimate failed.
- Meal confirmed.
- Photo deletion scheduled.

Never log:

- Full auth tokens.
- Stripe secrets.
- AI API key.
- Food photo raw payloads.
- Full request bodies.
- PII.

## Tests Required

Unit tests:

- Calorie target calculation.
- Today status calculation.
- Entitlement state mapping.
- Purchase options gating.
- AI response schema validation.

E2E tests:

- `GET /healthz`.
- Authenticated `GET /api/me`.
- Create profile.
- Create daily plan.
- Get Today view.
- Request photo upload URL.
- Create mocked meal estimate.
- Confirm meal.
- Entitlement-required failure.

Coverage:

- Follow organisation coverage standards.
- API route coverage should be comprehensive before production.

## Mocking Strategy

The API agent should support mocked integrations first:

- Mock auth validation.
- Mock entitlement.
- Mock AI estimate.
- Mock S3 upload URL.

Then replace mocks with real integrations behind interfaces.

This lets mobile development proceed without waiting for every external service.

## Deliverables

The spawn agent should produce:

- API service scaffold in `api/`.
- `GET /healthz`.
- Auth middleware.
- Entitlement guard.
- Input validation schemas.
- Profile and daily plan routes.
- Today view route.
- Photo upload URL route.
- Meal estimate route with mock implementation.
- Meal confirmation/edit/delete routes.
- Unit and E2E test skeleton.
- `.env.example` updates if needed.
- Notes on missing secrets or external dependencies.

## Done Criteria

The API/backend work is ready for mobile integration when:

- `npm run typecheck` passes.
- `npm test` passes.
- `npm run test:coverage` passes.
- API exposes the primary mobile contracts.
- Mobile can complete the mocked V1 flow against the API.
- Entitlement guard blocks meal logging for inactive users.
- Today view updates after meal confirmation.

