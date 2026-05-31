---
name: clipfactory
description: Scrape YouTube channels into short clips, stitch with a CTA, and schedule to social platforms via clip-factory.
version: 1.0.0
author: clip-factory
license: MIT
platforms: [linux, macos]
required_environment_variables:
  - name: CLIPFACTORY_KEY
    prompt: "Enter your clip-factory API key"
    help: "Generate one at https://clip-factory.app → Settings → Agent API key"
metadata:
  hermes:
    tags: [video, youtube, tiktok, social-media, scheduling, clip-factory]
    related_skills: [postiz, youtube-content]
---

## When to use

The user wants to make, schedule, or batch-manage short-form clips from YouTube channels. Typical asks:

- "Make 50 vids of @whoever and schedule one every 6 hours next week."
- "Scrape @channel and post one a day."
- "How many credits do I have left?"

If the user mentions clip-factory by name, or short-form / TikTok / YouTube Shorts batching with scheduling, this skill applies.

## Getting the API key

Every call needs `Authorization: Bearer $CLIPFACTORY_KEY`. The key lives on the **user's computer** — resolve it once, save it, and never ask again.

**Before the first request, resolve the key in this order:** the environment, then the saved file on disk.

```bash
KEY="${CLIPFACTORY_KEY:-$(cat ~/.clip-factory/key 2>/dev/null)}"
echo "${KEY:+found}"   # prints "found" if a key is already available
```

- **If it prints `found`** (Hermes provisioned `$CLIPFACTORY_KEY`, or the user saved it on a previous run), use it — do **not** ask. Make it available for this session: `export CLIPFACTORY_KEY="$KEY"`.
- **If it's empty**, ask the user once: *"Paste your clip-factory API key — generate one at https://clip-factory.app → Settings → Agent API key."* Then save it to their machine so no agent ever has to ask again:

  ```bash
  mkdir -p ~/.clip-factory
  printf '%s' 'cf_live_...' > ~/.clip-factory/key   # the value the user pasted
  chmod 600 ~/.clip-factory/key
  export CLIPFACTORY_KEY="$(cat ~/.clip-factory/key)"
  ```

  Then confirm: *"Saved to ~/.clip-factory/key — I won't ask again on this computer."*

The key is per-user and gated on an active subscription. `~/.clip-factory/key` is owner-only (chmod 600); never echo it back, copy it elsewhere, or commit it.

## Quick reference

- **Base:** `https://api.clip-factory.app/v1`
- **Auth:** `Authorization: Bearer $CLIPFACTORY_KEY`
- **Content-Type:** `application/json` for all writes.

Routes:

```
GET  /quota                                         → { remaining, monthly_remaining, topup_remaining, monthly_limit, plan, plan_name, period_resets_at, per_call_limit }
GET  /projects                                      → { projects: [{ id, name, has_cta, youtube_channel_handle, default_caption }] }
GET  /projects/:id                                  → { id, has_cta, cta_count, default_caption, youtube_channel_handle }
GET  /accounts                                      → { provider, accounts: [{ id, platform, username? }] }
POST /scrape    { project_id, handle, max, duration_s }   → { clip_ids, requested, scraped, inserted, duration_s }
GET  /clips     ?project_id=...&ids=a,b,c&status=...      → { clips: [{ id, status, error_message, scheduled_for, ... }] }
POST /stitch    { project_id, clip_ids }                  → 202 { stitched: [{ clip_id, status }], skipped }
POST /schedule  { project_id, clip_id, scheduled_for, caption?, account_ids? }
                                                          → { ok, clip_id, provider, scheduled_for, ... }
GET  /analytics ?project_id=...&since=7d                  → { project_id, period, oldest_sync_at, aggregate, posts }
```

Constraints:

- `max` is **1–100** per scrape call (hard cap regardless of plan).
- `duration_s` is **3–8** seconds.
- `scheduled_for` is ISO 8601, at least **60s in the future** and at most **90 days** out.
- No two clips within the same project can be scheduled within 60s of each other.
- `account_ids` is required when the user's scheduler is Post-Bridge. With Postiz, all connected accounts are used automatically.

Clip statuses (from `GET /clips`): `downloading` → `ready` → `stitching` → `ready` (after stitch) → `scheduled` → `posted`. `failed` at any stage means that single clip is dead — skip it, don't retry.

## Procedure: make N clips and schedule

1. **`GET /projects`** — confirm which project to use. Confirm `has_cta: true`. If the user has only one project, use it without asking.
2. **`GET /quota`** — if `remaining < N`, tell the user how many they have and offer to do that many instead, or to top up.
3. **`POST /scrape`** with `{ project_id, handle, max: N, duration_s: 3 }` (or whatever duration the user specified). Save the returned `clip_ids`.
4. **Poll `GET /clips?project_id=...&ids=<comma-separated>`** every 15 seconds. Continue when every clip is `status: ready` or `failed`. Don't poll more often than that — you'll waste tokens for no signal.
5. **`POST /stitch`** with `{ project_id, clip_ids: <ready ones only> }`. This returns 202; poll again until those clips are `status: ready` (stitched) or `failed`.
6. **`POST /schedule`** for each successfully stitched clip, spaced by the user's cadence. Default cadence if unspecified: **4 hours**. Default start time: **the next round hour at least 5 minutes from now**.
7. **Verify**: `GET /clips?project_id=...&ids=<all>` and report the count in `scheduled` status, plus the schedule window, back to the user.

## Procedure: check what's queued

- `GET /clips?project_id=...&status=scheduled` — what's currently in the schedule queue.
- `GET /clips?project_id=...&status=posted` — what's already out.

## Procedure: report performance / how clips are doing

When the user asks something like *"how are my clips doing?"*, *"performance update"*, *"top performers this week"*:

1. **`GET /analytics?project_id=...&since=7d`** (or `30d`, `90d`, or omit for all-time).
   `since` accepts shorthand (`7d`, `12h`, `2w`) or an ISO 8601 datetime.
2. Read `aggregate` for the headline: total views/likes/comments across all posts in the window.
3. Read `oldest_sync_at` and compare to now. If the oldest cached sync is more than **24 hours old**, prefix the report with a freshness note like *"data last synced X ago — open the clip-factory dashboard and hit Sync if you want fresh numbers."*
4. From `posts[]`, pick top performers by `totals.views` for a per-clip leaderboard. Each post has `per_account` showing the same clip's numbers per platform (TikTok / Instagram / YouTube / etc.) — surface the platform mix when relevant.
5. Reply with a tight summary: total views, top 3 clips by views, any clips with anomalous engagement (e.g. comment counts way above their views suggesting controversy / viral hits).

## Pitfalls

- **`subscription_inactive` (402):** Stop immediately. Tell the user their clip-factory subscription isn't active and to reactivate at https://clip-factory.app/billing. Don't retry.
- **`insufficient_credits` (402):** Read `remaining` and `required` from the error body. Offer the user (a) do fewer clips, or (b) top up. Don't silently shrink the request — confirm with the user first.
- **`cta_missing` (409):** This project has no CTA video uploaded. You **cannot upload a CTA via the agent** — it's web-only. Tell the user to go to https://clip-factory.app, select the project, and upload a CTA, then retry.
- **`validation_failed` with `reason: "cadence_too_tight"`:** Two scheduled times are within 60s. Widen the cadence and retry that specific clip.
- **`validation_failed` with `reason: "too_soon"`:** Pushed below the 60s lead. Bump the time and retry.
- **`clip_not_ready` (409):** Clip hasn't finished downloading. Wait and poll `GET /clips` until status is `ready`, then retry the schedule call.
- **`clip_failed` items in a batch:** Normal — yt-dlp occasionally fails on a video. Skip and continue. Never abort the whole job for one bad clip. Credits are auto-refunded for failures.
- **`youtube_unavailable` (502):** The channel returned no shorts or is rate-limited / private. Tell the user. Don't retry within the same session.
- **`upstream_failed` (502):** Postiz / Post-Bridge call failed. Check the `provider` field. Tell the user and stop — don't retry blindly because the user's social account may need re-auth.
- **`rate_limited` (429):** Back off for `retry_after_seconds` then continue. This is purely defensive against runaway loops; in normal use you should never hit this.
- **Postiz scheduling** uses all of the user's connected social accounts automatically. **Post-Bridge** requires explicit `account_ids`: call `GET /accounts` first; if `provider: "postbridge"`, use every returned account id by default (unless the user specified which ones they want).
- **Never** call `/scrape` with `max > 100`. It will be capped server-side but you'll waste tokens generating the request.
- **Analytics is read-only and possibly stale.** `GET /analytics` returns whatever the dashboard last synced from Postiz/Post-Bridge. There is no agent-callable sync endpoint — refreshing happens when the user clicks Sync in the dashboard. Always check `oldest_sync_at` and warn the user if it's >24h old.

## Verification

Before declaring a multi-clip job done:

1. `GET /clips?project_id=...&ids=<all the ids you got back from /scrape>`.
2. Count statuses. Report to the user: "N scheduled between Monday 9am and Friday 9pm, M failed during download (refunded), K stitched but not yet scheduled."
3. If anything is still `stitching` or `downloading`, say so — don't claim completion.

## Notes

- Credits are atomic. If you scrape 50 but only 47 actually download, you're charged for 47, not 50. No need to manually refund.
- Use the project's `default_caption` when the user didn't supply one; the server uses the clip's YouTube title as a final fallback.
- `GET /clips` returns clips sorted newest first.
