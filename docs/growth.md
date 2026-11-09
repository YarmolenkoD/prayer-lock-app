# Growth and product notes

Numbers below are from the app's own analytics (PostHog) and the public App Store search API, late 2026. Shared because they are the kind of thing indie developers rarely publish.

## Activation: the first prayer is the leak, not sign-in

Sixty days of iOS and iPadOS users, autumn 2026:

| Step | Users | Share |
|---|---|---|
| Opened the app | 249 | 100% |
| Signed in (Apple, Google, email or guest) | 236 | 95% |
| Started a prayer | 179 | 72% |
| **Completed a prayer** | **28** | **16%** |

Sign-in was never the problem. Of those who started a first prayer, the median cancel came **15 seconds** after start (p90 at 40 seconds), and 79 people started and then just left the app. Almost everyone who completed tapped "I Prayed" early, around one minute in.

The first prayer started straight from the "All set" screen with the profile's daily duration, ten minutes by default. Ten minutes of silence on a timer, on first open, with Instagram locked, is a lot to ask.

Changes shipped:

- the first prayer is fixed at **2 minutes**;
- cancellation events carry elapsed seconds, planned minutes, origin and progress, so we can see where in the session people leave;
- a `prayer_session_backgrounded` event fires once per session;
- on Android, missing blocking permissions never prevent a prayer from starting.

A side effect of the old flow: the paywall came only after the first completed prayer, so about 84% of users never saw it. That explained a month with zero sales better than any pricing theory. The paywall now comes before the first prayer.

Measurement trap: `sign_up` fires only for email. Apple, Google and guest sign-ins log as `sign_in` with a `method`. Low `sign_up` counts do not mean low registrations.

## iPad is real

About **45%** of users are on iPad, with real iPad screen sizes (1180×820 landscape most common), spread across US cities. The app is universal, portrait-only, and its iPad layout had not been checked. Completion on iPad was 12% versus 18% on iPhone. If you build a "phone" app, look at your tablet share before assuming.

## ASO: the name is crowded

Scanning the public iTunes Search API across storefronts:

- About fifteen US apps are titled "Prayer Lock…", several of them Muslim prayer-time blockers. We ranked #14 for "prayer lock" and #17 for "quiet time" at the time of the scan.
- Same-niche rivals with thousands of ratings own every generic English term (prayer, bible, devotional, faith, screen time, app blocker). Long-tail is the only way in.
- German, French and Arabic prayer terms are dominated by Muslim prayer-time apps. Christian-specific words (Andacht, chrétien, صلاة مسيحية) are where the space is.
- Low-competition storefronts: Ukraine for every term, German "Andacht" and "Glaube", Mexican "rezar" and "católico", Korean "교회" and "기도", Russian "блокировка приложений".
- We had **zero ratings** in every storefront. The in-app rating prompt had reached one person out of 57 who completed a prayer, because the completion screen cancelled the prompt for non-subscribers to show the paywall instead. Check that your rating prompt actually fires.

## Positioning

Every rival in the Christian app-blocker space gates **every app open** behind a short prayer. Prayer Lock protects **one scheduled, unhurried prayer session a day**, and "I Prayed" ends it. That sentence is on the site, the listing, the directories and the comparison pages, written the same way each time, because AI search engines cross-check facts across sources and inconsistent claims lose citations.

## SEO and GEO for a tiny site

[prayer-lock.com](https://prayer-lock.com/) is static HTML on GitHub Pages. What turned out to matter:

- structured data (SoftwareApplication, Person for the founder, Article authors), hreflang across eight locales, a sitemap, `robots.txt` that allows AI crawlers, and an `llms.txt` with the facts and prices;
- comparison pages against each named competitor, because that is what people and models actually ask ("X vs Y", "alternative to X");
- consistent facts across App Store, site, Wikidata, AlternativeTo, SaaSHub, PeerPush, Product Hunt and this repository;
- an "about" page with a real name. Faceless apps get fewer citations.

Do not use a generic web search tool's `site:` query to check indexing. It is not Google. Use Search Console.

## Distribution channels tried

- Directories with free queues: AlternativeTo, SaaSHub, PeerPush, Uneed. Weeks to months of waiting, paid skip-the-line offers everywhere. Worth it for the backlinks and AI citations, not for traffic.
- Quora: six answers with disclosure and no links in one day is the ceiling for a new account. Two or three a week after that.
- Reddit: comment first, earn karma, post in builder subreddits before faith subreddits, share blog posts rather than store links.
- Product Hunt: scheduled for a Tuesday–Thursday with three landscape cards and a maker comment.
- Custom product pages and product page optimization tests in App Store Connect, driven by scripts (see [Shipping](shipping-ios.md)).

Related: [Subscriptions](subscriptions.md) for the pricing side of the funnel.
