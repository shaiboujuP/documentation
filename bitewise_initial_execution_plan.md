# Bitewise: Initial Execution Plan

## Product Summary

Bitewise is a photo-first calorie tracking app.

The V1 product loop is intentionally small:

1. Sign in with Google or Apple ID.
2. Complete onboarding.
3. Start a 14-day trial.
4. Take or upload a food photo.
5. Receive an AI calorie estimate.
6. Confirm or edit the meal.
7. See today's meal log and remaining calories.

V1 does not include a dashboard. The first version uses a simple Today view only.

## Confirmed Product Decisions

Auth:

- Google authentication.
- Apple ID authentication.
- No email/password auth in V1.

Billing:

- Stripe Billing on the web.
- 14-day free trial.
- $10/month after trial.

Hosting:

- AWS.

Photo retention:

- Delete meal photos 30 days after processing.
- Confirmed meal records stay after photo deletion.

App name:

- Bitewise.

## V1 User Experience

### Landing

Message:

> Track calories from a photo. Stay on top of your daily diet.

Primary CTA:

> Start free trial

### Authentication

Supported providers:

- Google
- Apple ID

### Onboarding

Collect:

- Age
- Gender
- Height
- Weight
- Goal: lose, maintain, gain
- Activity level
- Target pace: relaxed, standard, aggressive
- Optional dietary restrictions

Output:

> Your daily target is X calories.

The user can accept or manually adjust the target.

### Trial and Billing

Flow:

1. User completes onboarding.
2. User sees paywall.
3. User starts 14-day Stripe trial.
4. Backend stores subscription state.
5. User receives access to meal logging.

### Today View

V1 replaces the idea of a full dashboard with a simple Today view.

Show:

- Daily calorie target.
- Calories consumed today.
- Calories remaining today.
- Meals logged today.
- One simple status message.

Example statuses:

- "850 calories remaining today."
- "250 calories left today."
- "180 calories over today."

Do not include in V1:

- Full analytics dashboard.
- Weekly charts.
- Macro dashboard.
- Weight progress dashboard.
- Behavioral scoring.

### Add Meal

Flow:

1. Tap Add Meal.
2. Take photo or upload image.
3. Upload photo to AWS storage.
4. API starts AI estimate.
5. User reviews result.
6. User confirms, edits, or retakes.
7. Meal appears in Today view.

Estimate fields:

- Meal name.
- Estimated calories.
- Optional protein, carbs, fat.
- Detected food items.
- Confidence: low, medium, high.

## Initial Architecture

```text
bitewise/
  mobile/
  web/
  api/
  billing/
```

## Programming Language Per Folder

| Folder | Primary language | Runtime/framework | Responsibility |
| --- | --- | --- | --- |
| `mobile/` | TypeScript | Expo React Native | iOS and Android app |
| `web/` | TypeScript | React 19 with Vite | Landing, login, onboarding support, Stripe checkout |
| `api/` | TypeScript | Node.js on AWS | App API, meal logic, AI estimation, storage coordination |
| `billing/` | TypeScript | Node.js on AWS | Stripe checkout, customer portal, webhooks, entitlement sync |

## AWS Target Architecture

Recommended V1 AWS services:

- Amazon S3 for meal photo storage.
- Amazon RDS for Postgres.
- AWS ECS Fargate or Lambda for the API.
- AWS Lambda for Stripe webhooks and scheduled cleanup.
- Amazon EventBridge Scheduler for photo deletion jobs.
- Amazon CloudWatch for logs and alarms.
- AWS Secrets Manager for API keys and Stripe secrets.
- Amazon CloudFront if serving uploaded/static assets through a CDN.

## Data Model

Minimum tables:

- `users`
- `profiles`
- `daily_plans`
- `subscriptions`
- `meals`
- `meal_items`
- `food_photos`
- `notification_preferences`
- `analytics_events`

Important rules:

- A user has one active profile.
- A user has one active daily plan.
- A user must have an active trial or paid subscription to log meals.
- Food photos are deleted 30 days after processing.
- Meal records remain after photo deletion.
- Dates should be stored with user timezone awareness.

## AI Meal Estimation

AI flow:

1. API receives uploaded photo reference.
2. API or worker calls a vision-capable model.
3. Model returns structured JSON.
4. API validates the JSON.
5. API stores a meal estimate.
6. User confirms or edits the estimate.

Example AI output shape:

```json
{
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

Validation rules:

- Calories must be positive.
- The response must match the schema.
- Low-confidence estimates can still be shown to the user.
- The user must always be able to edit calories before confirming.

## V1 Metrics

Keep metrics small:

- `registration_completed`
- `onboarding_completed`
- `trial_started`
- `first_meal_logged`
- `meal_confirmed`
- `meal_estimate_edited`
- `meal_estimate_failed`
- `subscription_canceled`
- `trial_converted`

V1 success signal:

> A meaningful percentage of trial users log their first meal on day 0.

## V1 Alerts

Keep alerts minimal:

- One optional meal logging reminder.
- One close-to-limit message after confirming a meal.
- One optional end-of-day summary.

No complex notification campaign in V1.

## Main Risks

AI estimate trust:

- Mitigate with editable estimates, confidence labels, and clear approximate wording.

Payment flow:

- Keep billing isolated in `billing/`.
- Validate Stripe webhook signatures.
- Treat Stripe as the payment source of truth.

AWS complexity:

- Start with simple managed services.
- Avoid custom infrastructure unless needed.

Trial drop-off:

- Optimize onboarding and first meal logging before adding analytics or dashboards.

Privacy:

- Enforce the 30-day photo deletion policy from the first release.

## First Build Target

The first vertical slice should be:

1. Google or Apple sign-in.
2. Onboarding.
3. Daily calorie target.
4. Stripe trial checkout.
5. Add a meal with mocked AI.
6. Confirm meal.
7. Today view updates.

After that works end to end, replace mocked AI with real photo estimation.
