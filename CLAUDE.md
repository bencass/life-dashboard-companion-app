# Life Dashboard Companion App

Android app that reads health data from **Health Connect**, packages it as JSON, and POSTs it to a Supabase edge function once per hour. Source for the `daily_health` table on the Life Monitor dashboard.

- **Package:** `com.owen282000.lifedashboard`
- **Current version:** 1.2.1 (`versionCode = 4`)
- **Upstream:** https://github.com/owen282000/life-dashboard-companion-app (forked; Ben's fork at https://github.com/bencass/life-dashboard-companion-app)
- **Dashboard repo:** `../../life tracker/life_monitor` (Next.js + Supabase)
- **Supabase project ref:** `fugmajnyemsszzqgjqem` (region: Sydney)

## Data pipeline

```
Garmin watch (+ Strava for activities, separate pipeline)
  → Garmin Connect app
  → Android Health Connect (local store)
  → this companion app (WorkManager PeriodicWorkRequest, 60 min default)
  → POST JSON to Supabase edge function `health-sync`
  → upsert into `daily_health` (+ `hr_samples`) via `upsert_daily_health` RPC
     (which computes residual-active server-side)
  → Life Monitor dashboard reads `daily_health`
```

Activities (runs, rides, gym) are a separate pipeline: Garmin → Strava → Strava webhook → `strava-webhook` edge function → `activities` table. Not this app's concern.

Relevant source files:

| File | Purpose |
|---|---|
| `app/src/main/java/com/owen282000/lifedashboard/HealthConnectManager.kt` | Reads all 23 data types from Health Connect. Daily-total metrics (steps, distance, active calories, total calories) use `AggregateRequest` with AEST day boundaries. Everything else uses `ReadRecordsRequest`. |
| `app/src/main/java/com/owen282000/lifedashboard/HealthSyncManager.kt` | Builds the JSON payload and posts via `WebhookManager`. Advances per-type `lastSync` timestamps in `SharedPreferences` after a successful POST. |
| `app/src/main/java/com/owen282000/lifedashboard/HealthSyncWorker.kt` | `CoroutineWorker` invoked by WorkManager. |
| `app/src/main/java/com/owen282000/lifedashboard/LifeDashboardApplication.kt` | Schedules `HealthSyncWorker` as `PeriodicWorkRequest`. Interval from `PreferencesManager.getHealthSyncIntervalMinutes()`, default 60. |
| `app/src/main/java/com/owen282000/lifedashboard/PreferencesManager.kt` | All persistent app state: webhook URLs/headers, enabled data types, per-type last-sync timestamps, sync interval. |
| `app/src/main/java/com/owen282000/lifedashboard/WebhookManager.kt` | OkHttp POST with retries + logs each call to the Logs tab. |

## Battery optimisation — MUST BE UNRESTRICTED

**Settings → Apps → Life Dashboard → Battery → Unrestricted.**

`PeriodicWorkRequestBuilder` is best-effort under Android Doze / App Standby. If the app is on the default "Optimised" bucket, Android can defer the 60 min sync indefinitely — observed **6+ hour gaps on 2026-04-20** (last sync 10:13 AEST, then nothing until manual intervention at 16:30 AEST), which looks identical to an overwriting-values bug on the dashboard but is really just the phone not talking.

Changed to **Unrestricted on 2026-04-20**. If syncs start lagging again, re-check this setting first — it's the single most likely cause of stale data.

Secondary mitigations (code-side, not yet implemented):
- Request `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` during onboarding so the user doesn't have to dig into settings.
- Consider a foreground service with a persistent notification for syncing.
- Surface "Last sync: Nm ago" on the app's home screen so staleness is visible before opening the dashboard.

## Known issues

- **DNS resolution failure** — observed once on 2026-04-19 21:05 AEST. Transient network / Cloudflare. The retry logic in `WebhookManager` (3 attempts, exponential backoff) recovered on the next scheduled sync. If it recurs, check the phone's DNS settings / VPN.
- **HTTP 500 on large payloads** — observed 2026-04-19 18:02 AEST with a 1041-record payload. The edge function likely timed out or hit a memory limit processing that many HR samples in one call. Mitigations: periodic syncs should keep record counts small, but a long offline period (no sync overnight + all day) could reload a big backlog. If this recurs, consider chunking `hr_samples` inserts on the edge function side (currently 500-row batches inside `processDate`, but the overall function still does one RPC per date).

## Calorie model — IMPORTANT

**Do not trust `ActiveCaloriesBurnedRecord` from Garmin's Health Connect write.** Garmin only writes active-calorie records for explicitly recorded workouts, so the Health Connect aggregate for ACTIVE_CALORIES is usually a small fraction of Garmin Connect's own all-day figure (Apr 19 2026: Garmin UI said 550 kcal, Health Connect aggregate was 153 kcal).

`TotalCaloriesBurnedRecord` (ENERGY_TOTAL) is reliable — it's Garmin's all-day total (BMR + active).

Residual-active is computed **in the Supabase `upsert_daily_health` RPC / trigger**:

```
residual_active = max(0, total_calories − BMR × fractionOfAestDay)
```

- `fractionOfAestDay = 1.0` for any date before today (AEST)
- `fractionOfAestDay = (seconds since AEST midnight) / 86400` for today
- `BMR` is Mifflin–St Jeor over `user_profile.date_of_birth / height_cm / sex / weight_kg`
- Clamp to `≥ 0` (Garmin under-reports early in the morning; a small negative is just math, not a deficit)

Both values land in `daily_health`: raw `total_calories` + derived `active_calories`.

**Do not replicate this math in TS** (the edge function) — it already happens in the DB and a TS duplicate would double-count. The edge function just forwards raw Garmin totals to `p_total_calories` and the raw Garmin active value to `p_active_calories`; the DB decides what ends up in the column.

Companion app's job is just to deliver both raw arrays; it does no BMR math.

See: `../../life tracker/life_monitor/supabase/functions/health-sync/index.ts` (v9+) and the `upsert_daily_health` function definition on the Supabase project `fugmajnyemsszzqgjqem`.

## Timezone

Everything is AEST (`Australia/Sydney`, UTC+10, no DST handling needed).

- Companion app day-bucket loop: `LocalDate.ofInstant(..., ZoneId.of("Australia/Sydney")).atStartOfDay(sydneyZone).toInstant()`. Payload `start_time` / `end_time` are serialized as UTC `Instant.toString()`.
- Edge function recovers the AEST date by shifting UTC +10h then slicing `.toISOString().split("T")[0]`.

Both sides must agree. If you add a new daily-aggregated metric, match the existing pattern in `readStepsData` / `readActiveCaloriesData`.

## Webhook payload shape

Top-level keys: `timestamp`, `app_version`, `source`, plus one array per enabled data type (only included when non-empty). Aggregated types use `{calories|count|meters, start_time, end_time}` per day; instantaneous types use `{value..., time}`.

Full reference: see `README.md` §Webhook Payload Format.

## Gotchas discovered during debugging

- **`readRecords()` returns a random subset, not daily totals.** Fixed in commit `1165d64` by switching steps/distance/active_calories/total_calories to `aggregate()`. If you're ever tempted to read daily totals via `readRecords()` + a manual sum — don't. Duplicate intervals from multiple sources will give wildly inaccurate numbers (e.g. step count bouncing between 35 and 3816).
- **`lastSync` filter in aggregate readers uses `effectiveEnd >= lastSync`.** Once `lastSync` passes midnight AEST, the previous day's entry (whose `effectiveEnd` is exactly the next-day 00:00 AEST) stops getting re-sent. In practice this is fine for the server's calorie model (residuals for past days don't change once total is stored), but be aware if you change how `lastSync` advances.
- **Garmin Connect → Health Connect writes fragmentary ACTIVE_CALORIES.** See the calorie-model section above. This is a Garmin-side limitation, not a bug in either repo.
- **Sleep stage ints vs strings.** Health Connect exposes `stage` as `Int` constants; the edge function expects lower-case strings like `"deep"`, `"rem"`. Mapping lives in `HealthConnectManager.sleepStageToString()`. See commit `cc4a2a3`.
- **`HealthSyncManager.buildJsonPayload()` only emits an array when the list is non-empty.** This keeps payloads slim but means the edge function must tolerate missing keys (it does — uses `payload.foo ?? []` throughout).

## Building

- Debug APK: `./gradlew assembleDebug` → `app/build/outputs/apk/debug/app-debug.apk`
- GitHub Actions: `.github/workflows/build.yml` builds a debug APK on push to `main` and uploads as an artifact.
- Release signing is not configured; sideload the debug APK.

## Supabase-side contracts to honor

If you change the payload schema, update the edge function at the same time. Specifically:

- Field renames on aggregated types: both sides look for `calories` (active, total) / `count` (steps) / `meters` (distance), with `start_time`/`end_time`.
- New daily-aggregated types: follow the `readActiveCaloriesData` pattern (AEST day-bucket loop + `AggregateRequest`) and add the type to `collectDates()` in the edge function so its dates get processed.
