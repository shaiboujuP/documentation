# Bitewise: Mobile Development Repo Agent Brief

## Purpose

This document is for a spawn agent that will start mobile development in the Bitewise repo.

Repo:

```text
/Users/shai/git/bitewise
```

Mobile folder:

```text
/Users/shai/git/bitewise/mobile
```

## Non-Negotiable Standards

Follow:

- `/Users/shai/git/documentation/DEVELOPMENT_GUIDELINES.md`
- `/Users/shai/git/documentation/CICD_GUIDELINES.md`

Key rules:

- TypeScript only.
- Node 22.x.
- npm only.
- Strict TypeScript.
- ESM.
- React hooks only.
- No Tailwind.
- Plain React Native styles.
- Do not commit secrets.
- Work from `dev`.
- Open PRs toward `dev` for feature work unless instructed otherwise.

## Mobile Scope For V1

Build the mobile app around one loop:

1. Sign in.
2. Complete onboarding.
3. Start trial through an available purchase option.
4. Add a meal from a photo.
5. Review AI estimate.
6. Confirm or edit meal.
7. See today's meals and remaining calories.

No full dashboard in V1.

Use a simple Today view.

## Required Mobile Architecture

Recommended structure:

```text
mobile/
  src/
    app/
    auth/
    onboarding/
    billing/
    meals/
    today/
    camera/
    api/
    shared/
    tests/
      unit/
      e2e/
  app.config.ts
  eas.json
  package.json
  tsconfig.json
```

Module responsibilities:

- `auth/`: Google and Apple ID auth UI and session handoff.
- `onboarding/`: age, gender, height, weight, goal, activity, target pace.
- `billing/`: purchase option screen, in-app purchase adapter, web link-out adapter.
- `meals/`: estimate review, confirm/edit/delete meal.
- `today/`: calorie target, consumed, remaining, meal list.
- `camera/`: image capture, upload preparation.
- `api/`: typed API client.
- `shared/`: mobile-safe types and utilities.

## Purchase Options

The mobile repo must support two purchase options in code.

### Option A: In-App Purchase

iOS:

- Apple In-App Purchase.
- Native subscription purchase.
- Restore purchases.

Android:

- Google Play Billing.
- Native subscription purchase.
- Restore purchases.

Implementation rule:

- Keep native purchase code behind an interface.
- API should verify purchases server-side.
- Mobile should not decide entitlement alone.

Suggested interface:

```ts
export interface PurchaseOption {
  id: "iap" | "web_linkout";
  label: string;
  available: boolean;
  unavailableReason?: string;
}

export interface PurchaseResult {
  provider: "apple" | "google" | "stripe";
  status: "started" | "completed" | "canceled" | "failed";
}
```

### Option B: Link Out To Web

Behavior:

- Mobile asks API which purchase options are available.
- If web link-out is available, app opens a web checkout URL.
- Stripe checkout happens on web.
- App refreshes entitlement after return.

Rules:

- Must be feature-flagged.
- Must be region/programme gated.
- Must use store-compliant copy.
- Must not appear globally by default.

## API Contracts Needed

Mobile should expect typed endpoints:

```text
GET /api/me
POST /api/profile
POST /api/daily-plan
GET /api/today
GET /api/purchase-options
POST /api/purchases/iap/verify
POST /api/purchases/web-checkout-session
POST /api/photos/upload-url
POST /api/meal-estimates
POST /api/meals
PATCH /api/meals/:id
DELETE /api/meals/:id
```

If an endpoint does not exist yet:

- Create a typed mock layer.
- Do not block UI development.
- Keep mock data close to the final contract.

## Screens To Build First

1. Welcome/auth screen.
2. Onboarding flow.
3. Calorie target confirmation.
4. Purchase options screen.
5. Today view.
6. Add Meal camera/upload screen.
7. Estimate review screen.
8. Meal edit screen.
9. Settings/account screen.

## UX Rules

- V1 should feel fast and quiet.
- No marketing landing screen inside the app after install.
- No full dashboard.
- Today view is the home screen.
- Use supportive, non-judgmental copy.
- Make estimates visibly approximate.
- Always allow calorie editing before confirmation.

## Testing Requirements

Add tests for:

- Onboarding target payload creation.
- Today view calorie math rendering.
- Purchase option selection behavior.
- Link-out disabled state.
- Meal estimate edit form.
- API error state handling.

Manual checks:

- iOS simulator launch.
- Android emulator launch.
- Camera permission state.
- Photo picker fallback.
- Offline/error states.

## Deliverables

The spawn agent should produce:

- Mobile app scaffold in `mobile/`.
- Scripts in `mobile/package.json`.
- Typed navigation/app shell.
- Mocked V1 flow.
- Purchase option interfaces.
- API client skeleton.
- Tests for core logic.
- Notes on any native-module or Expo limitations.

