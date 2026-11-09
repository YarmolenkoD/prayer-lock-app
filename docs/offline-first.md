# Offline-first sync

People pray in places without signal: planes, basements, retreats, the countryside. A prayer that finishes offline must count, keep the streak, and appear in history immediately. Prayer Lock's sync layer has three parts, all in `src/lib/offline/` and all backed by MMKV.

## 1. Snapshot cache

A per-user mirror of server data, keyed `snapshot:{userId}:{domain}`. Domains: profile, recent sessions, analytics (week, month, year), notification settings, the first page of prayers.

Every read is **cache-first**: paint from the snapshot, refresh from the network, fall back to the snapshot if the network fails. Lists merge in pending outbox items so an offline prayer never disappears from history while it waits to upload. Snapshots are cleared on sign-out.

## 2. Outbox

A durable FIFO queue of mutations (`offline_outbox_v1`), scoped by user id. Kinds:

- `complete_prayer`: upsert with a **client-generated UUID** so retries are idempotent (`ignoreDuplicates`)
- `save_mood`
- `delete_session`
- `update_profile` (last write wins)
- `update_notification_settings`
- `upsert_onboarding`

Error handling splits in two:

- **Retryable** (network, 5xx, 429, 401, PostgREST connection errors): stop the flush, keep the queue intact, preserve order.
- **Permanent** (validation, RLS denial): drop the item and report to Sentry.

The outbox **survives sign-out**. If the same user signs back in, pending items resume. It is only cleared on account deletion. This was a deliberate choice: losing a prayer because someone signed out on a plane is worse than holding a few rows for a user who never returns.

## 3. Sync orchestrator

`initSync()` runs from the root layout and watches `NetInfo` plus `AppState`. `syncNow()` flushes the outbox, then refreshes profile and prayer data only when something was actually flushed and nothing remains. A small Zustand store drives the offline banner and a sign-out warning in settings ("you have 2 unsynced prayers").

## Streaks are computed on the server

Originally the client computed streak and total-prayer counts and wrote them to the profile. With offline sessions arriving hours or days late, that produced wrong streaks and a "heal to 1" hack.

Now:

- `prayer_sessions.local_date` stores the user's local calendar date (`YYYY-MM-DD`) at completion time, so a prayer finished offline at 23:50 counts for that day even if it uploads tomorrow.
- A Postgres function `recalc_profile_stats(uid)` recomputes streak and totals from sessions, and a trigger on `prayer_sessions` calls it on insert, update and delete.
- The client still computes optimistically for instant UI, and still enqueues `update_profile` with the same numbers as a fallback for older rows.
- The outbox has a fallback for the migration window: on `PGRST204` or `42703` (column missing) it strips `local_date` and retries, so old clients and new servers coexist.
- Stale-streak reset is display-only. No more client-side writes to fix streaks.

We trust the device clock. Anti-cheat is not a goal for a prayer app.

## Rollout

The migration ran against production in July 2026 with a one-time backfill of every profile. Divergence between client-computed and server-computed stats after the heal: zero.

## If you copy this

- Generate ids on the client. Retries are then free.
- Separate retryable from permanent errors and never reorder the queue.
- Keep the outbox across sign-out. Clear it on delete.
- Move derived aggregates (streaks, totals) to the database once offline writes exist. The client cannot know what else arrived.
