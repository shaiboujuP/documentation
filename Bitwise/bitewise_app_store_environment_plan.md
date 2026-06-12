# Bitewise: App Store Environment Plan

## Goal

Prepare everything needed to bring Bitewise to TestFlight and then the Apple App Store.

## Key App Store Risk

Bitewise sells access to digital app functionality through a subscription.

Apple's current App Review Guidelines say that apps unlocking features or functionality through subscriptions generally must use In-App Purchase. Stripe Billing is approved for the web path, but using Stripe as the only purchase path inside the iOS app may cause App Store rejection unless Bitewise qualifies for a specific external purchase entitlement.

Recommended V1 decision:

- Use Stripe Billing on web.
- Build the mobile entitlement layer to support two purchase paths:
  - Option A: in-app purchase.
  - Option B: link out to web checkout where allowed.
- For safest App Store approval, support Apple In-App Purchase for iOS subscriptions.
- Keep link-out behind a remote feature flag and region/programme gate.

Official references:

- Apple In-App Purchase: https://developer.apple.com/in-app-purchase/
- Apple App Review Guidelines: https://developer.apple.com/app-store/review/guidelines/
- Apple subscriptions: https://developer.apple.com/app-store/subscriptions/
- Expo iOS submit: https://docs.expo.dev/submit/ios/

## App Identity

Recommended identifiers:

- App name: Bitewise.
- Internal app code: `bitewise`.
- iOS bundle identifier: `com.bitewise.app`.
- Android application ID: `com.bitewise.app`.
- App Store Connect SKU: `bitewise-ios`.
- Stripe product code: `bitewise-premium-monthly`.

## Required Accounts

Apple:

- Apple Developer Program membership.
- App Store Connect access.
- Admin or App Manager permissions.
- App Store Connect API key for CI submission.

Expo:

- Expo account.
- EAS project.
- EAS token for CI.

Google:

- Google Cloud project for OAuth.
- iOS OAuth client.
- Web OAuth client.

Stripe:

- Stripe account.
- Test mode.
- Live mode.
- Product and price.
- Webhook endpoint.
- Customer portal configuration.

AWS:

- AWS account.
- Sandbox environment.
- Production environment.
- IAM roles for CI/CD.
- Secrets Manager.
- RDS, S3, runtime service, logs.

## Apple Developer Setup

1. Enroll in Apple Developer Program.
2. Create App ID:
   - Bundle ID: `com.bitewise.app`.
   - Enable Sign in with Apple.
   - Enable push notifications if V1 notifications ship.
3. Create App Store Connect app record:
   - Name: Bitewise.
   - SKU: `bitewise-ios`.
   - Bundle ID: `com.bitewise.app`.
4. Create App Store Connect API key:
   - Store issuer ID.
   - Store key ID.
   - Store `.p8` key securely.
5. Configure EAS credentials:
   - Distribution certificate.
   - Provisioning profile.
   - Push notification key if notifications ship.

## Expo/EAS Setup

Required files in `mobile/`:

- `app.json` or `app.config.ts`.
- `eas.json`.
- Native bundle ID set to `com.bitewise.app`.
- Build profiles:
  - development
  - preview
  - production

Required secrets:

- `EXPO_TOKEN`
- `EXPO_APPLE_ID` if needed
- `ASC_API_KEY_ID`
- `ASC_API_ISSUER_ID`
- `ASC_API_KEY_P8`

Build flow:

1. `eas build --platform ios --profile preview`
2. Submit to TestFlight.
3. Test with internal users.
4. Fix review-blocking issues.
5. Build production profile.
6. Submit App Store review.

## App Store Listing Requirements

Prepare:

- App name.
- Subtitle.
- Description.
- Keywords.
- Category.
- Age rating.
- Support URL.
- Marketing URL.
- Privacy policy URL.
- Terms of service URL.
- Screenshots for required device sizes.
- App preview video, optional.
- Review notes.
- Demo account if login is required.

Nutrition-specific copy:

- State that calorie estimates are approximate.
- Avoid medical claims.
- Avoid promising weight loss outcomes.
- Do not position the app as clinical nutrition treatment.

## Privacy Requirements

Data collected:

- Account identifier.
- Profile inputs: age, gender, height, weight, goal.
- Meal records.
- Food photos.
- Subscription status.
- Analytics events.

Policy:

- Meal photos deleted 30 days after processing.
- Meal records remain unless the user deletes their account.
- No secrets or PII in logs.
- User must be able to request account deletion.

App Store privacy labels need to reflect:

- User content.
- Health/fitness-related data if applicable.
- Purchases/subscription data.
- Identifiers.
- Usage data.

## Payment Environment

Web Stripe:

- Test product and price.
- Live product and price.
- Checkout success URL.
- Checkout cancel URL.
- Customer portal.
- Webhook endpoint.
- Webhook signing secret.

iOS/mobile subscription options:

Option A, in-app purchase:

- Apple In-App Purchase for iOS.
- Google Play Billing for Android.
- Native purchase sheet.
- Store-managed trial/subscription lifecycle.
- Server-side receipt/transaction verification.

Option B, link out to web:

- Stripe Checkout on web.
- External link shown only where allowed.
- Remote feature flag controls visibility.
- Region/programme eligibility must be enforced.
- Link-out copy must be store-compliant.

Option C, account access only:

- Mobile app allows login to an existing paid account.
- No in-app purchase button.
- No checkout link.
- Lower conversion, lower review risk than unsafe link-out.

Recommended path:

- Build Stripe web billing now.
- Build provider-agnostic entitlements now.
- Implement Apple IAP before App Store submission if selling from inside iOS.
- Implement Google Play Billing before Google Play production if selling from inside Android.
- Keep link-out disabled by default until eligibility is confirmed.
- Stripe, Apple, and Google should all unlock the same `premium` access.

## AWS Environment

Sandbox:

- API URL: `https://api.sandbox.bitewise...`
- Billing webhook URL: `https://billing.sandbox.bitewise...`
- S3 bucket for sandbox photos.
- RDS sandbox database.
- Stripe test mode secrets.
- Google OAuth sandbox redirect URIs.

Production:

- API URL: `https://api.bitewise...`
- Billing webhook URL: `https://billing.bitewise...`
- S3 bucket for production photos.
- RDS production database.
- Stripe live mode secrets.
- Google OAuth production redirect URIs.

Required secrets:

- `DATABASE_URL`
- `AWS_REGION`
- `S3_PHOTO_BUCKET`
- `STRIPE_SECRET_KEY`
- `STRIPE_WEBHOOK_SECRET`
- `GOOGLE_CLIENT_ID`
- `APPLE_CLIENT_ID`
- `AI_API_KEY`
- `GRAFANA_LOGS_URL`
- `GRAFANA_LOGS_API_KEY`
- `GRAFANA_API_KEY`

## CI/CD Requirements

Per org guidelines:

- GitHub Actions.
- `dev` branch deploys to sandbox.
- `master` branch deploys to production.
- Docker images pushed to ECR.
- Smoke test calls `/healthz`.
- Production deploy happens only through PR merge into `master`.

Mobile CI additions:

- EAS build for preview.
- EAS submit to TestFlight after release branch or production workflow.
- Store credentials are secrets, never repo files.

## TestFlight Plan

Internal TestFlight:

- Founder/product owner.
- Engineering.
- QA.

External TestFlight:

- 10-50 early users.
- Focus on first meal logging and estimate trust.

Beta test checklist:

- Sign in with Google.
- Sign in with Apple ID.
- Onboarding.
- Trial/subscription entitlement.
- Add meal photo.
- Confirm/edit estimate.
- Today view update.
- Photo deletion job scheduled.
- Account deletion route exists.

## Submission Checklist

Before App Store review:

- App has no broken links.
- Privacy policy URL works.
- Terms URL works.
- Demo account works if required.
- App does not link to Stripe checkout unless allowed.
- Subscription/paywall path is compliant.
- Approximate nutrition disclaimer is visible.
- Food photos retention policy is documented.
- Push notification permission copy is clear if notifications ship.
- Crash-free TestFlight build.

## Timeline

Week 1:

- Apple Developer account and App Store Connect setup.
- Expo/EAS setup.
- Bundle ID and app record.

Week 2:

- First preview iOS build.
- TestFlight internal testing.
- Auth provider configuration.

Week 3:

- Payment compliance decision.
- Stripe web billing validated.
- Apple IAP plan if needed.

Week 4:

- External TestFlight.
- App Store metadata.
- Privacy labels.
- Submission rehearsal.

Week 5:

- Final production build.
- Submit for App Store review.
