# Subscriptions with RevenueCat

Prayer Lock is paid: $39.99 a year with a 7-day free trial, or $7.99 a month (US prices). No ads, no data sale. Payments go through RevenueCat with `react-native-purchases`. These are the mistakes worth not repeating.

## Package ids are `$rc_annual` and `$rc_monthly`

RevenueCat's standard package identifiers start with `$rc_`. Code that looked up `$annual` and `$monthly` found nothing, so the offering never matched and users saw "Unable to load subscription options". Use the typed accessors on the current offering (`current.annual`, `current.monthly`) with `packageType` as a fallback, and match prices by product-id substring. Never hardcode a package id.

## The entitlement identifier is the identifier, not the display name

Our entitlement is displayed as "Prayer Lock Pro" and its identifier is also `Prayer Lock Pro`, which is not what the code assumed (`premium`). A permissive fallback masked the mismatch for weeks. Copy the identifier field exactly.

Also attach every product to the entitlement in the dashboard. Device logs showing `activeSubscriptions = ['prayerlock_yearly']` with `entitlements.active = {}` mean the product is not attached. As a safety net the access check is `!!entitlement || activeSubscriptions.length > 0`, so a live store subscription grants access even if the dashboard is misconfigured.

## Entitlements are device-based, not account-based

The first design tied the subscription to the Supabase account with `Purchases.logIn(userId)`. We reversed it: purchases belong to the **RevenueCat anonymous id**, which is restorable through the App Store account. Reasons:

- guests can buy without creating an account;
- switching Supabase accounts on one device does not lose the purchase;
- it matches how users think about App Store purchases.

The entitlement check runs at app launch, not on auth changes, and a `CustomerInfo` listener keeps it live.

Reinstall gap: deleting and reinstalling creates a fresh anonymous id with no entitlement. On launch, if not subscribed, the app calls `syncPurchases()`, which silently re-posts the device receipt; RevenueCat transfers the subscription to the new id. Unlike `restorePurchases()`, it never shows an App Store sign-in prompt.

## Trial copy must be gated by eligibility

`introPrice` on a product is store configuration, not a per-user right. Apple grants an introductory offer **once per Apple ID per subscription group**, and upgrades never get one. A paywall that always says "7-day free trial" will promise a trial to people who are then charged full price. Gate all trial copy with `Purchases.checkTrialOrIntroductoryPriceEligibility`; show the trial only on `ELIGIBLE`. The Android SDK always returns `UNKNOWN`, so there we fall back to the presence of a free intro phase, which Play pre-filters per user.

## "Subscriptions unavailable" in production while TestFlight worked

Same binary, same RevenueCat config, empty offering in the App Store build only. Not a code bug.

Sandbox and TestFlight StoreKit serve **unapproved** products. Production StoreKit does not. The app had been approved without its subscriptions attached, so they stayed in "Prepare for Submission" and were invisible to paying users.

What to know:

- The **first** in-app purchase of an app has no standalone "Submit for Review" button. It must be attached to an app **version** and submitted together. Later products can go standalone.
- "Prepare for Submission" is the normal draft state. It does not change by filling fields; it changes when you submit with a version.
- Real blockers are per-section warnings: review screenshot (at least 640×920), price, duration, product and group localization.
- Check that the Paid Apps Agreement is Active. Sandbox ignores it, production needs it.

## Check the store, not your plan

Six months after launch an API check of App Store Connect showed the live prices and offers did not match what we believed had been set, and that the free trial did not exist on any product, which explained zero trial starts across dozens of new users. Script the check (`/v1/subscriptionGroups/{id}/subscriptions`, then `/introductoryOffers` and `/prices?filter[territory]=USA`) and run it after every store change.

## Smaller things

- Purchases cannot be tested on the iOS simulator. The "Purchase was cancelled" error with a long Metro stack is the SDK logging, not a crash. Use a device with a sandbox tester.
- App Store "cancel" turns off auto-renew; `isActive` stays true until `expiresDate`. Show it as active.
- Sandbox durations are compressed (a year is about an hour) and auto-renew stops after roughly twelve cycles. Paywalls reappearing fast in sandbox is expected.
- Guard `showPaywall` against stacking on itself. Check the current route before pushing.
- Manual premium grants for friends and testers live in a Postgres table matched on the JWT email, so they work without a build and without RevenueCat promotional entitlements.
- Price and offer experiments run as separate RevenueCat offerings selected by a PostHog feature flag payload, so a test can start and stop without a release.
