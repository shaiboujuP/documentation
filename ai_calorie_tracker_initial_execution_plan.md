# AI Calorie & Diet Tracker: Initial Execution Plan

## Product Summary

The app helps consumers track calories and daily food intake with minimal manual work. The core loop is:

1. Create an account.
2. Define a daily nutrition plan.
3. Take a photo of food.
4. Receive an estimated meal result.
5. Confirm or edit the result.
6. Track calories against the daily target.

The monetization model is a 14-day free trial followed by a $10/month subscription.

## Recommended Stack

Mobile:

- Expo React Native for iOS and Android.

Web:

- Next.js for landing page, account access, and dashboard review.

Backend:

- TypeScript API using Fastify or NestJS.
- Postgres database.
- Prisma ORM.
- S3-compatible storage for meal photos.

Authentication:

- Google sign-in.
- Apple sign-in.
- Email fallback.

AI estimation:

- Vision-capable OpenAI model.
- Structured JSON output.
- Server-side validation and calorie sanity checks.

Subscriptions:

- RevenueCat for cross-platform entitlement state.
- App Store billing on iOS.
- Google Play Billing on Android.
- RevenueCat Web Billing or Stripe for web.

Analytics:

- PostHog or Amplitude.

Notifications:

- Expo Notifications.
- APNs and FCM.

## Core Architecture

The product should be split into five major services:

1. Client apps
   - Mobile is the primary experience.
   - Web is useful for landing, account access, and dashboard review.

2. Backend API
   - Owns users, profiles, meal logs, daily plans, dashboard summaries, alerts, and subscription status.

3. AI estimation worker
   - Processes meal photos.
   - Calls the vision model.
   - Normalizes the result.
   - Produces editable meal estimates.

4. Billing and entitlement service
   - Receives subscription events.
   - Stores current access state.
   - Gates premium product access.

5. Notification scheduler
   - Sends reminders, warnings, daily summaries, and trial lifecycle messages.

## Data Model

Minimum tables:

- `users`
- `profiles`
- `daily_plans`
- `subscriptions`
- `meals`
- `meal_items`
- `food_photos`
- `daily_summaries`
- `notification_preferences`
- `analytics_events`

Important modeling notes:

- The billing provider remains the payment source of truth.
- The app database stores local subscription state for fast entitlement checks.
- Meal dates should be timezone-aware.
- Photos should be deletable without deleting confirmed meal records.

## AI Meal Logging Flow

1. User taps `Add Meal`.
2. User takes or uploads a food photo.
3. Backend creates a `food_photos` record.
4. Worker sends the image to the vision model.
5. Model returns meal name, food components, calories, macros, and confidence.
6. Backend validates and normalizes the response.
7. App shows an editable confirmation screen.
8. User confirms, edits, or retakes.
9. Confirmed meal updates the daily dashboard.

The app should always describe estimates as approximate.

## User Flows

### Landing

Message:

> Track calories from a photo. Stay on top of your daily diet.

Primary CTA:

> Start free trial

### Registration

Supported methods:

- Google
- Apple
- Email fallback

### Onboarding

Collect:

- Age
- Gender
- Height
- Weight
- Goal
- Activity level
- Target pace
- Optional dietary restrictions

Output:

> Your daily target is X calories.

### Paywall

Show after onboarding and before full product usage.

Offer:

- 14 days free.
- Then $10/month.
- Cancel anytime.

### Dashboard

Show:

- Daily calorie target.
- Calories consumed.
- Calories remaining.
- Meals logged today.
- Simple status.

### Meal Confirmation

Actions:

- Confirm.
- Edit.
- Retake.

## Build Phases

### Phase 0: Product and Design

Duration: 1 week.

Deliverables:

- Clickable prototype.
- Final MVP scope.
- Core screen designs.
- Event taxonomy.
- AI result schema.
- Nutrition disclaimer language.

### Phase 1: App Foundation

Duration: 1-2 weeks.

Deliverables:

- Monorepo.
- Mobile app shell.
- Web app shell.
- Backend API shell.
- Database setup.
- Auth setup.
- CI.
- Error tracking.

### Phase 2: Onboarding and Plan Setup

Duration: 1 week.

Deliverables:

- Registration.
- Profile form.
- Calorie target calculation.
- Manual target override.
- Dashboard shell.

### Phase 3: Subscription Gate

Duration: 1-2 weeks.

Deliverables:

- Paywall.
- RevenueCat integration.
- Entitlement checks.
- Webhook handling.
- Restore purchases.

### Phase 4: Photo Meal Logging

Duration: 2 weeks.

Deliverables:

- Camera/upload.
- Image storage.
- AI worker.
- Structured meal estimates.
- Edit/confirm flow.
- Meal history.

### Phase 5: Dashboard and Alerts

Duration: 1-2 weeks.

Deliverables:

- Daily calorie math.
- Status messages.
- Push notifications.
- Daily summary.
- Trial nudges.

### Phase 6: Web MVP

Duration: 1 week.

Deliverables:

- Landing page.
- Login.
- Dashboard.
- Account/subscription page.

### Phase 7: QA and Beta

Duration: 2 weeks.

Deliverables:

- Payment sandbox testing.
- TestFlight build.
- Android internal test build.
- AI evaluation set.
- Push notification testing.
- Analytics validation.
- Privacy review.

### Phase 8: Launch

Duration: 1 week.

Deliverables:

- Store listings.
- Production billing.
- Monitoring dashboards.
- Support flow.
- Launch checklist.

## MVP Acceptance Criteria

The product is launchable when a user can:

1. Create an account on mobile.
2. Complete onboarding.
3. Start a 14-day free trial.
4. Take a food photo.
5. Receive an estimate.
6. Confirm or edit the meal.
7. See calories update on the daily dashboard.
8. Receive useful reminders.
9. Access the same account from web.
10. Convert to paid subscription after the trial.

## Testing Plan

Unit tests:

- Calorie target calculation.
- Dashboard math.
- Entitlement logic.

Integration tests:

- Auth.
- Meal creation.
- AI worker.
- Billing webhooks.
- Notification scheduling.

End-to-end tests:

- Onboarding.
- Paywall.
- Add meal.
- Edit meal.
- Dashboard update.

Payment tests:

- Apple sandbox.
- Google Play test tracks.
- Web checkout test mode.
- Restore purchases.

AI tests:

- 100-200 common meal photos.
- Estimate latency.
- Malformed output rate.
- Confidence distribution.
- User edit distance.

## Main Risks

AI accuracy:

- Mitigate with editable estimates, confidence labels, conservative wording, and sanity checks.

Subscription complexity:

- Use RevenueCat to centralize entitlement state.

Trial drop-off:

- Optimize Day 0 first meal logging and Day 1-3 retention.

AI cost:

- Compress images, avoid duplicate calls, cache estimates, and monitor cost per confirmed meal.

Privacy:

- Decide whether meal photos are retained or deleted after processing.

## Launch Metrics

Track:

- `registration_completed`
- `onboarding_completed`
- `paywall_viewed`
- `trial_started`
- `first_meal_logged`
- `meal_confirmed`
- `meal_edited`
- `meals_per_user_per_day`
- `day_1_retention`
- `day_3_retention`
- `day_7_retention`
- `trial_converted`
- `subscription_canceled`

North-star MVP metric:

> Users who log at least two meals per day on three separate days during the first week.

