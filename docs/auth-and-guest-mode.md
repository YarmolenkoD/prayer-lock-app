# Auth and guest mode

## The rejection

In June 2026 Apple rejected the app under **Guideline 5.1.1(v)**: the app required creating an account before using its core feature. Blocking apps on your own device does not need an account, so Apple is right.

## Guest mode with Supabase anonymous auth

The fix was a "Continue without an account" button backed by `supabase.auth.signInAnonymously()`.

Why anonymous auth instead of a local-only mode:

- Anonymous users are **full Supabase users**. The `handle_new_user` trigger creates a profile row (email nullable). Row-level security works because anonymous sessions get the `authenticated` role with an `is_anonymous` JWT claim. No second code path for "local data".
- The offline outbox, sync and analytics all keep working unchanged.

Upgrading a guest to a permanent account happens **in place**: `signUpWithEmail` checks `is_anonymous` and calls `updateUser({ email, password, data })` instead of creating a new user. Same user id, all sessions and streaks kept. The profile screen shows a guest banner with "Sign in or create account" instead of editable fields, and sign-out as a guest shows a data-loss warning in all ten locales.

Known gap: signing in with Apple or Google from a guest session replaces the session rather than linking it. The proper fix is `linkIdentity` with manual linking enabled; not done yet.

Operational notes that were not obvious:

- "Allow anonymous sign-ins" is a dashboard toggle. Nothing works until it is on.
- GoTrue rejects `example.com` addresses as invalid. Use a real domain for test accounts.
- Supabase's built-in SMTP allows roughly two emails per hour. Configure custom SMTP before launch or sign-ups and guest upgrades will fail with a rate-limit error at the worst time.

## Native Google sign-in

Google sign-in moved from the browser-based OAuth flow to native `@react-native-google-signin/google-signin` plus `supabase.auth.signInWithIdToken`, because the browser flow showed users a "Continue to <project>.supabase.co" screen that looked like phishing.

Two settings in Supabase's Google provider are required and easy to miss:

- **Skip nonce checks: ON.** The iOS Google SDK embeds a nonce in the id token but does not expose it, so Supabase cannot compare it. With the check on you get "Passed nonce and nonce in id_token should either both exist or not".
- **Authorized Client IDs** must list both the iOS and the web client id, or you get "Unacceptable audience".

The web client secret is not used by the app and should not be in the app's `.env`. It belongs only in the Supabase dashboard.

Because `ios/` is a committed bare project, the Google config plugin does not run; the reversed client id URL scheme was added to `Info.plist` by hand.

## Why this matters for a prayer app

Guests are about a fifth of sign-ins. Asking for an email before someone has prayed once is the wrong order, the same way asking for a subscription before the first prayer was the wrong order (see [Growth](growth.md)). Let people pray first.
