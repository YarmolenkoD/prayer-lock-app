# Privacy

Short version: Prayer Lock never sees what you pray. The full policy is at [prayer-lock.com/privacy.html](https://prayer-lock.com/privacy.html).

## On-device

- App blocking uses Apple's Screen Time frameworks on iOS and an overlay shield on Android. Which apps you chose to block is stored on your device (iOS keeps the selection as an opaque token the app cannot even read as names).
- The prayer timer, the shield and the "I Prayed" button run on the device.
- Snapshot cache and the offline outbox are stored in MMKV on the device.

## Stored on the server (Supabase)

- Your account (email if you created one; nothing if you continue as a guest).
- Prayer sessions: when, how long, planned duration, whether you finished early, your local date.
- A mood you optionally pick after a session.
- Notification settings and onboarding answers (prayer time, goals).
- Streak and totals, derived from sessions.

Every row is protected by row-level security tied to your user id, including guest users.

## Never collected

- The content of your prayers. There is no text field for them and no microphone use.
- What you do in blocked apps.
- Contacts, location, photos.

## Third parties

- **RevenueCat** for subscriptions. Identified by an anonymous id, not by your email.
- **Sentry** for crash reports.
- **PostHog** for product analytics (events like "prayer started", "paywall shown"). Used to improve the app, not sold.
- **Apple / Google** for payments and sign-in if you choose them.

## Deleting

Delete Account in Settings removes your account and all server data in one call, and clears local storage. Instructions are also at [prayer-lock.com/delete-account.html](https://prayer-lock.com/delete-account.html).

## Funding

Prayer Lock is funded only by subscriptions. There are no ads and no data sale, and there is no plan for either.
