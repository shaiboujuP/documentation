# Bitewise: Mobile Development Environment Needs

## Purpose

This document is for a spawn agent that will prepare or validate the local mobile development environment for Bitewise.

The mobile app lives in:

```text
/Users/shai/git/bitewise/mobile
```

The repo must follow:

- `/Users/shai/git/documentation/DEVELOPMENT_GUIDELINES.md`
- `/Users/shai/git/documentation/CICD_GUIDELINES.md`

## Confirmed Mobile Direction

- Mobile framework: React Native.
- Recommended toolchain: Expo.
- Language: TypeScript.
- Package manager: npm only.
- Node version: 22.x from root `.nvmrc`.
- App code: `bitewise`.
- iOS bundle ID: `com.bitewise.app`.
- Android application ID: `com.bitewise.app`.

## Local Machine Requirements

Required:

- macOS for iOS development.
- Node.js 22.x.
- npm.
- Git.
- Xcode latest stable version.
- Xcode Command Line Tools.
- CocoaPods if a native dependency requires it.
- Watchman, recommended for React Native file watching.
- Expo CLI through local npm scripts or `npx`.
- EAS CLI through local npm scripts or `npx`.

Recommended checks:

```bash
node --version
npm --version
git --version
xcodebuild -version
xcrun simctl list devices
npx expo --version
npx eas --version
```

## Apple/iOS Requirements

Accounts:

- Apple Developer Program account.
- App Store Connect access.
- Permission level: Admin, App Manager, or equivalent build/submission capability.

Identifiers:

- Bundle ID: `com.bitewise.app`.
- App Store Connect SKU: `bitewise-ios`.

Capabilities:

- Sign in with Apple.
- Push Notifications if V1 reminders ship.
- In-App Purchase if selling from inside iOS.

Secrets/credentials:

- App Store Connect API key ID.
- App Store Connect issuer ID.
- App Store Connect `.p8` private key.
- Apple Team ID.
- EAS token.

Do not commit credentials.

## Android Requirements

Required:

- Android Studio.
- Android SDK.
- Android emulator.
- Java version compatible with the selected Expo SDK.

Identifiers:

- Android application ID: `com.bitewise.app`.

Capabilities:

- Google Sign-In.
- Push notifications if V1 reminders ship.
- Google Play Billing if selling from inside Android.

## Environment Variables

Mobile local development should read only public/mobile-safe values directly.

Expected mobile-safe values:

```text
EXPO_PUBLIC_API_BASE_URL=
EXPO_PUBLIC_WEB_BASE_URL=
EXPO_PUBLIC_GOOGLE_IOS_CLIENT_ID=
EXPO_PUBLIC_GOOGLE_ANDROID_CLIENT_ID=
EXPO_PUBLIC_GOOGLE_WEB_CLIENT_ID=
EXPO_PUBLIC_APPLE_SERVICE_ID=
EXPO_PUBLIC_ENABLE_IAP_PURCHASE=
EXPO_PUBLIC_ENABLE_WEB_CHECKOUT_LINKOUT=
```

Never put server secrets in mobile env files:

- Stripe secret key.
- Stripe webhook secret.
- AI API key.
- AWS secret key.
- Database URL.

## Purchase Environment

The mobile app must support two purchase options in architecture.

Option A: in-app purchase

- iOS: Apple In-App Purchase.
- Android: Google Play Billing.
- Preferred for store-distributed mobile subscription purchase.

Option B: link out to web

- Opens a Stripe Checkout web flow.
- Must be feature-flagged.
- Must be region/programme gated.
- Must not be enabled by default for all users.

Agent instruction:

- Build UI and interfaces so both options can exist.
- Keep actual availability controlled by remote config or API response.
- Do not hard-code link-out availability.

## Expo/EAS Environment

Expected files in `mobile/`:

```text
app.config.ts
eas.json
package.json
tsconfig.json
src/
```

Expected EAS profiles:

- `development`
- `preview`
- `production`

Commands to support:

```bash
npm install
npm run typecheck
npm test
npx expo start
npx eas build --platform ios --profile preview
npx eas build --platform android --profile preview
```

## Simulator/Device Testing

iOS:

- Test on at least one current iPhone simulator.
- Test on one physical iPhone before TestFlight if available.

Android:

- Test on one Pixel emulator.
- Test on one physical Android device if available.

Required mobile flows:

- Launch app.
- Google sign-in.
- Apple ID sign-in on iOS.
- Onboarding.
- Purchase option screen.
- Add Meal photo flow.
- Estimate review.
- Confirm/edit meal.
- Today view updates.

## Output Expected From Spawn Agent

The spawn agent should return:

- Whether the local environment is ready.
- Missing tools or credentials.
- Exact versions found.
- Any blockers for iOS simulator.
- Any blockers for Android emulator.
- Whether Expo/EAS commands are available.
- Recommended next fixes.

