# Life Dashboard Companion App

Android app that reads health data from **Health Connect**, packages it as JSON, and POSTs it to a Supabase edge function once per hour. Source for the `daily_health` table on the Life Monitor dashboard.

- **Package:** `com.owen282000.lifedashboard`
- **Current version:** 1.2.1 (`versionCode = 4`)
- **Upstream:** https://github.com/owen282000/life-dashboard-companion-app (forked; this worktree is Ben's fork)
- **Dashboard repo:** `../../life tracker/life_monitor` (Next.js + Supabase)

## Data pipeline

```
Garmin watch
  → Garmin Connect app
  → Android Health Connect (local store)
  → this companion app (WorkManager periodic, 60 min default)
  → POST JSON to Supabase edge function `health-sync`
  → upsert into `daily_health` (+ `hr_samples`) via `upsert_daily_health` RPC
  → Life Monitor dashboard reads `daily_health`
```

Relevant source files:

| File | Purpose |
|---|---|
| `app/src/main/java/com/owen282000/lifedashboard/HealthConnectManager.kt` | Reads all 23 data types from Health Connect. Daily-total metrics (steps, distance, active calories, total calories) use `AggregateRequest` with AEST day boundaries. Everything else uses `ReadRecordsRequest`. |
| `app/src/main/java/com/owen282000/lifedashboard/HealthSyncManager.kt` | Builds the JSON payload and posts via `WebhookManager`. Advances per-type `lastSync` timestamps in `SharedPreferences` after a successful POST. |
| `app/src/main/java/com/owen282000/lifedashboard/HealthSyncWorker.kt` | `CoroutineWorker` invoked by WorkManager. |
| `app/src/main/java/com/owen282000/lifedashboard/LifeDashboardApplication.kt` | Schedules `HealthSyncWorker` as `PeriodicWorkRequest`. Interval from `PreferencesManager.getHealthSyncIntervalMinutes()`, default 60. |
| `app/src/main/java/com/owen282000/lifedashboard/PreferencesManager.kt` | All persistent app state: webhook URLs/headers, enabled data types, per-type last-sync timestamps, sync interval. |
| `app/src/main/java/com/owen282000/lifedashboard/WebhookManager.kt` | OkHttp POST with retries + logs each call to the Logs tab. |

## Calorie model — IMPORTANT

**Do not trust `ActiveCaloriesBurnedRecord` from Garmin's Health Connect write.** Garmin only writes active-calorie records for explicitly recorded workouts, so the Health Connect aggregate for ACTIVE_CALORIES is usually a small fraction of Garmin Connect's own all-day figure (Apr 19 2026: Garmin UI said 550 kcal, Health Connect aggregate was 153 kcal).

`TotalCaloriesBurnedRecord` (ENERGY_TOTAL) is reliable — it's Garmin's all-day total (BMR + active).

So the edge function derives the active figure on the server:

```
residual_active = max(0, total_calories − BMR × fractionOfAestDay)
```

- `fractionOfAestDay = 1.0` for any date before today (AEST)
- `fractionOfAestDay = (seconds since AEST midnight) / 86400` for today
- `BMR` is Mifflin–St Jeor over `user_profile.date_of_birth / height_cm / sex / weight_kg`
- Clamp to `≥ 0` (Garmin under-reports early in the morning; a small negative is just math, not a deficit)

Both values land in `daily_health`: raw `total_calories` + derived `active_calories`. Companion app's job is just to deliver both raw arrays; it does no BMR math.

See: `../../life tracker/life_monitor/supabase/functions/health-sync/index.ts` (v9+).

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
