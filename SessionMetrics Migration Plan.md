---
title: SessionMetrics Migration Plan
type: design
tags:
  - backend
  - firebase
  - shared
  - performance
  - critical
created: 2026-04-11
aliases:
  - Flat Metrics Migration
  - Quantized Metrics Plan
---

# SessionMetrics Migration Plan

A design doc for introducing a new `sessionMetrics/{sessionId}` collection that holds every per-session sports metric as a **flat queryable top-level field**, written by the phone atomically with `sensorData`, updated by the Cloud Function with cross-session metrics, and used as the primary data source for the website + Agent Mode.

> [!info] Why this exists
> Today the website's calendar and coach console pay for the full `sensorData` payload (5–10 MB per 90-min session) even when they only need a handful of summary numbers. The `derived.stats` nested object can't be queried with Firestore `where()` clauses, so every "find me all sessions with maxSpeed > 28" becomes a client-side filter after a full scan. This note describes how to fix both problems at once without breaking any existing client.

---

## Current state (verified 2026-04-11)

### What the phone computes before upload

`Cresento/src/src/utils/StatsEngine.ts:computeStats()` (~1100 lines) produces a `SessionAnalytics` object with:

```ts
{
  stats: SessionStats {
    totalSteps,
    distanceStepsMeters, distanceIntegratedMeters,
    walkingDistanceMeters, joggingDistanceMeters, sprintDistanceMeters,
    maxSpeedKmh, averageSprintSpeedKmh, maxAcceleration,
    minutesPlayed,
    percentStanding, percentWalking, percentJogging, percentSprinting,
    sprintEvents, decelEvents,
    caloriesBurned, averageStepFrequencyHz,
    fatigueIndex,         // null if <20 min
    fatigueScore,         // 1–10, null if <20 min, from sprint speed ratio
    avgSprintSpeedFirst20Kmh, avgSprintSpeedLast20Kmh
  },
  speed: SpeedPoint[],    // 20 Hz raw series, used for charts
  topSpeedMoments, timeline, phases, sprintInsight, accelInsight,
  intervalRawData
}
```

Called from `SessionUploader.ts:uploadSession()` (line 142) which writes ONLY `derived.stats` and `derived.speed` to `sensorData/{id}`. Everything else (`topSpeedMoments`, `timeline`, `phases`, `intervalRawData`) is **discarded** after the write.

### What the Cloud Function adds AFTER upload

`functions/src/index.ts:processAndSaveAnalytics()` (lines 970–1109) waits for `derived.stats` + `derived.speed` to be present on the new `sensorData/{id}` doc, then computes and writes:

**To `sensorData/{id}.precomputedAnalytics`** (top-level field, lightweight):
- `accelerationSummary` — count, maxAccel, minAccel, avgPos/Neg, explosiveAccelerations, hardDecelerations
- `regression` — slope, intercept, rSquared, speedDeclinePct, speedLoss
- `performance` — intensityIndex, fatigueIndex (zone-based 0–100), avgRecoveryTime, consistencyScore, staminaDecay, peakWindow
- `highIntensity` — hsrDistance, sprintDistance, hsrCount, sprintCount
- `sessionMetrics` — cadence, peakCadence, cadenceVariability, workRate, highIntensityDistance, sprintFrequency, avgSprintDuration, explosiveAccelerations, totalDistanceM, durationMin

**To `sensorData/{id}/additional_data/analytics`** (subcollection doc, heavy):
- Full `acceleration: Array<{t,a}>` — re-derived from speed because the phone discarded it
- `segmentedFatigue` — the 8-ratio weighted fatigue calculation
- `distanceBreakdown` — hsrDistanceM, highAccelDistanceM, highDecelDistanceM, explosiveDistanceM, hmlDistanceM
- `eventCounts` — highAccelEvents, highDecelEvents, hsrEvents

**Separately on a daily schedule** (`calculateDailyACWR`, line 1434):
- `precomputedAnalytics.acwr` — value, injuryRisk, acuteLoad, chronicLoad — written to the most recent session per player based on rolling 7/28 day windows

### Where `derived.stats` actually lives in practice

It's **buried inside a nested object** in a multi-page document. To query "all sessions where the player hit > 28 km/h", the website has to:

1. Download every `sensorData/{id}` metadata doc (filters by `userID` or `uploadedAt` only — can't push the metric comparison into Firestore)
2. Parse each one's `derived.stats.maxSpeedKmh`
3. Filter client-side

Same for every other metric. There's no server-side index.

---

## Target state

### New collection: `sessionMetrics/{sessionId}`

- **Same doc ID** as the matching `sensorData/{sessionId}` so joins are trivial (`sessionMetrics/${id}` and `sensorData/${id}` are always parallel)
- **Flat top-level fields** — every metric queryable via `where()` and `orderBy()`
- **Small doc** — ~2 KB target, well under the 1 MB Firestore limit, no pages subcollection needed
- **Primary read source** for the website calendar, coach console, agent mode overview tools, leaderboard views, and anything else that needs metadata without raw time-series
- **Written atomically** by the phone in the same batch as `sensorData/{id}` during upload
- **Updated by the Cloud Function** for the cross-session metrics that genuinely need server-side compute (ACWR, regression, segmentedFatigue)
- **sensorData remains** the sole home for raw time-series data (`derived.speed`, `pages/*`). Only touched when a tool actually needs those bytes.

### Schema (TypeScript)

```ts
/**
 * sessionMetrics/{sessionId}
 *
 * One doc per session. Same ID as sensorData/{sessionId}. Small, flat,
 * queryable. Written by the phone, updated by the Cloud Function.
 */
export interface SessionMetricsDoc {
  // ── Identity & provenance ─────────────────────────────────────────────
  sessionId: string              // redundant with doc ID, kept for easy listing
  userId: string                 // Firebase Auth UID of the wearer
  orgId: string | null
  playerId: string | null        // players/{id} ref if linked
  teamId: string | null
  sessionType: "game" | "training" | "session"
  schemaVersion: 1               // bump when we change the shape

  // ── Temporal ──────────────────────────────────────────────────────────
  uploadedAt: Timestamp          // server timestamp at upload
  sessionStartTime: Timestamp    // wall-clock session start
  sessionEndTime: Timestamp      // wall-clock session end
  durationMin: number            // (end - start) / 60, pre-computed

  // ── Speed (flat, queryable) ───────────────────────────────────────────
  maxSpeedKmh: number
  avgSpeedKmh: number            // total distance / duration in km/h
  avgSprintSpeedKmh: number | null  // null if zero sprints
  avgSprintSpeedFirst20Kmh: number | null  // for fatigue #1
  avgSprintSpeedLast20Kmh: number | null

  // ── Distance (meters, flat) ───────────────────────────────────────────
  totalDistanceM: number
  sprintDistanceM: number
  hsrDistanceM: number
  walkingDistanceM: number
  joggingDistanceM: number

  // ── Distance per minute (pre-normalized) ─────────────────────────────
  // Eliminates the "is this a 30-min training or a 90-min game" problem
  // that's been biting the agent mode.
  distancePerMinM: number
  sprintDistancePerMinM: number
  hsrDistancePerMinM: number

  // ── Explosiveness ─────────────────────────────────────────────────────
  sprintEvents: number
  highAccelEvents: number
  highDecelEvents: number
  sprintEventsPerMin: number
  highAccelEventsPerMin: number
  highDecelEventsPerMin: number

  // ── Activity zones (percentages sum to 100) ───────────────────────────
  percentSprinting: number
  percentJogging: number
  percentWalking: number
  percentStanding: number

  // ── Steps & cadence ───────────────────────────────────────────────────
  totalSteps: number
  cadence: number                // steps/min (= averageStepFrequencyHz * 60)
  averageStepFrequencyHz: number
  caloriesBurned: number
  maxAcceleration: number

  // ── Fatigue (renamed to unambiguous names) ────────────────────────────
  // Today three different "fatigue" fields with different scales and
  // directions exist on the same session. These new names make the
  // direction and scale explicit so callers can't mix them up.
  //
  // sprintSpeedFatigueScore1to10: from StatsEngine.ts, 1–10, HIGHER = MORE fatigued
  //   Formula: bucket(avgSprintSpeedLast20 / avgSprintSpeedFirst20)
  //   1 = sprints collapsed, 10 = sustained peak sprint speed
  //   ⚠️ Direction is backwards from the current `fatigueScore` label — today
  //   the tooling shows "higher = more fatigued" but the formula produces
  //   higher = less fatigued. THIS MIGRATION FIXES IT by explicitly flipping:
  //   sprintSpeedFatigueScore1to10 = 11 - rawFatigueScore  (so 10 = most tired)
  sprintSpeedFatigueScore1to10: number | null
  sprintSpeedFatigueConfidence: number | null

  // activityLoadIndex0to100: from Cloud Function performance.fatigueIndex
  //   Formula: min(100, sprinting% × 1.5 + jogging% × 0.8)
  //   Higher = MORE ACTIVITY (NOT fatigue — the old name was misleading).
  //   Essentially a workload intensity proxy.
  activityLoadIndex0to100: number | null

  // segmentedFatigueScore0to10: from analytics-utils.ts, 0–10
  //   Formula: 8-ratio weighted average, end window vs peak rolling window
  //   10 = no decline from peak, 0 = total collapse
  //   ⚠️ Direction: HIGHER = LESS fatigued (opposite of sprintSpeedFatigueScore1to10)
  //   Kept on this scale for backward compat with any UI reading it. To
  //   convert to "higher = more fatigued": 10 - segmentedFatigueScore0to10
  segmentedFatigueScore0to10: number | null
  segmentedFatigueConfidence: number | null

  // ── Workload (sessionMetrics — currently in precomputedAnalytics) ─────
  workRateMPerMin: number         // = totalDistanceM / durationMin
  sprintFrequency: number         // sprints per 10 minutes
  avgSprintDuration: number       // seconds per sprint event
  explosiveAccelerations: number  // count >= 2.0 m/s²
  hardDecelerations: number       // count <= -2.0 m/s²

  // ── Speed regression (cloud-computed, may be null initially) ─────────
  speedRegressionSlope: number | null
  speedRegressionIntercept: number | null
  speedRegressionRSquared: number | null
  speedDeclinePct: number | null
  speedLoss: number | null
  staminaDecay: number | null
  consistencyScore: number | null

  // ── ACWR (cloud-computed daily schedule, null until daily job runs) ──
  acwrValue: number | null
  acwrInjuryRisk: "High" | "Moderate" | "Low" | "Optimal" | null
  acwrAcuteLoad: number | null
  acwrChronicLoad: number | null
  acwrAcuteDays: number | null
  acwrChronicDays: number | null
  acwrComputedAt: Timestamp | null

  // ── Metadata ──────────────────────────────────────────────────────────
  metricsComputedAt: Timestamp
  // Which system most recently wrote to this doc — lets us tell at a
  // glance if a given session is still missing its cloud-computed fields
  metricsLastSource: "phone" | "cloud_function"
}
```

**Size estimate:** ~1.5 KB per doc uncompressed. ~50 fields × ~20 bytes each + Firestore overhead. Fits comfortably inside any batched read.

### Read savings

| Operation | Before (`sensorData`) | After (`sessionMetrics`) | Speedup |
|---|---:|---:|---:|
| Coach calendar — list 500 sessions with basic metrics | ~25 MB (500 × ~50 KB summaries) | ~0.75 MB (500 × ~1.5 KB) | **33×** |
| Coach console — 50 full sessions | ~250 MB (50 × 5 MB with raw speed) | ~75 KB (50 × 1.5 KB) | **3,300×** |
| Agent `query_team_overview` 2-metric scan of 50 players | ~2.5 MB | ~75 KB | **33×** |
| Agent `query_game_metrics` 10-player game | ~50 MB (10 × 5 MB) | ~15 KB (10 × 1.5 KB) | **3,300×** |

Latency drops by the same ratio. Calendar page load from ~4 seconds to ~120 ms.

### Query unlock

New queries become trivial that today require full-scan-then-filter:

```ts
// "Who on the team broke 28 km/h in the last 14 days?"
db.collection("sessionMetrics")
  .where("orgId", "==", orgId)
  .where("uploadedAt", ">=", twoWeeksAgo)
  .where("maxSpeedKmh", ">=", 28)
  .orderBy("maxSpeedKmh", "desc")
  .limit(20)

// "Sessions where the player accumulated > 500 m of sprint distance"
db.collection("sessionMetrics")
  .where("userId", "==", uid)
  .where("sprintDistanceM", ">", 500)
  .orderBy("sprintDistanceM", "desc")

// "Anyone in the amber/red ACWR zone right now"
db.collection("sessionMetrics")
  .where("teamId", "==", teamId)
  .where("acwrValue", ">", 1.3)
```

Firestore needs composite indexes for these — the migration script will emit the index definitions for `firestore.indexes.json`.

---

## Implementation phases

### Phase A — Write the new collection from the phone

**File: `Cresento/src/src/utils/SessionUploader.ts`**

1. Add a new helper `buildSessionMetricsDoc(stats, metadata, sessionId): SessionMetricsDoc` that takes the existing `SessionStats` from `computeStats()` and the upload metadata and produces a flat doc matching the schema above.
2. In `uploadSession()`, after computing `stats` and before the `sensorData` write, build the `sessionMetrics` doc.
3. Use a Firestore **batch write** to atomically write both `sensorData/{id}` and `sessionMetrics/{id}` so they can't drift.
4. The phone populates every field EXCEPT the cloud-computed ones (`speedRegressionSlope*`, `acwr*`, `segmentedFatigue*`). Those stay null until the Cloud Function updates them.
5. Set `metricsLastSource: "phone"` on every phone-written doc.

**Key rule:** The phone ALSO continues writing `sensorData.derived.stats` and `derived.speed` exactly as today. This migration is **additive** — no existing read path breaks. We kill `derived.stats` reads later after every client has been migrated to `sessionMetrics`.

### Phase B — Update the Cloud Function to write cross-session metrics to `sessionMetrics`

**File: `functions/src/index.ts`**

1. After `processAndSaveAnalytics()` computes `precomputedAnalytics` for a sensorData doc, ALSO update the matching `sessionMetrics/{id}` doc with:
   - `speedRegressionSlope` ← `precomputedAnalytics.regression.slope`
   - `speedRegressionRSquared` ← `precomputedAnalytics.regression.rSquared`
   - `speedDeclinePct`, `speedLoss`
   - `staminaDecay`, `consistencyScore`
   - `segmentedFatigueScore0to10` ← `additional_data/analytics.segmentedFatigue.score`
   - `segmentedFatigueConfidence`
   - `activityLoadIndex0to100` ← `precomputedAnalytics.performance.fatigueIndex`
   - `metricsLastSource: "cloud_function"`, `metricsComputedAt: serverTimestamp()`
   Use `update()` not `set()` so the phone-written fields are preserved.
2. Similarly in `calculateDailyACWR` (line 1434), write the ACWR result to `sessionMetrics/{latestSessionId}` in addition to the current `precomputedAnalytics.acwr` field.
3. **Delete** the redundant Cloud Function compute of `sessionMetrics` (cadence/workRate/etc.) — the phone writes those directly now. The existing block in `processAndSaveAnalytics()` lines ~1030 onward can be stripped to JUST the cross-session stuff (regression, segmented fatigue, ACWR). Cuts Cloud Function runtime by ~60%.

### Phase C — Migrate existing sensorData docs

**File: `scripts/backfill-sessionMetrics.mjs`** (new)

One-time migration that walks every `sensorData/{id}` doc, extracts the metrics, and writes a matching `sessionMetrics/{id}` doc. Idempotent — running it twice just re-writes the same values.

Pseudocode:
```
for each sensorData doc:
  if sessionMetrics/{id} already exists: skip
  extract phone-computable fields from derived.stats
  extract cloud-computable fields from precomputedAnalytics
  extract segmentedFatigue from additional_data/analytics subcol if present
  write sessionMetrics/{id} in batched commits of 400
```

Run in dry mode first, review, then `--yes` for the real backfill.

### Phase D — Swap website reads to `sessionMetrics`

**Files: `Cresento Website/Cresento.net/lib/firestore.ts`**

Add new functions:
- `getSessionMetrics(sessionId)` — returns `SessionMetricsDoc | null`
- `getSessionMetricsByOrg(orgId, opts)` — with date range + type filter
- `getSessionMetricsByPlayer(userId, opts)`
- `getSessionMetricsByTeam(teamId, opts)`
- `listSessionMetricsForCalendar(orgId, monthRange)` — replaces the current full `sensorData` scan

Migrate existing call sites ONE AT A TIME:
- Sessions calendar → `listSessionMetricsForCalendar`
- Coach console → `getSessionMetricsByTeam` + direct metric field reads
- Agent Mode tools → see Phase E

Keep the old `getSensorData*` functions alive for chart rendering and the agent mode sandbox (`analyze_with_code`, `get_session_timeseries`).

### Phase E — Swap Agent Mode tools to `sessionMetrics`

**File: `lib/agent/tools.ts`**

- `query_team_overview` → read from `sessionMetrics` directly (flat fields mean no more `SUMMARY_STAT_MAP` extractor table, just direct field access)
- `query_game_metrics` → same
- `query_session_metrics` → read `sessionMetrics/{id}` directly
- `query_metric_summary` → can now use Firestore `where()` + `orderBy()` for trends instead of full-scan-then-sort
- `compare_players` → parallel `getSessionMetricsByPlayer` calls, much smaller payload
- `load_game_data` → unchanged, still needs `sensorData` roster info
- `get_session_timeseries` → unchanged (still reads raw from `sensorData`)
- `analyze_with_code` → unchanged (still needs raw for full-resolution math)

Also: update `lib/agent/metric-docs.ts` to reflect the new unambiguous fatigue field names and drop the "label bug" gotchas since the migration fixes them.

### Phase F — Update the RN app's read paths

**File: `Cresento/src/src/...`** (various)

If the RN app also displays per-session metrics to players, migrate it to read from `sessionMetrics` for the same speed gains. Calendar views, leaderboards, progress pages.

The RN app currently writes `sessionMetrics` (Phase A) but doesn't read it yet.

### Phase G — Deprecate `derived.stats`

After every client has been switched over:
1. Stop writing `derived.stats` in `SessionUploader.ts` (keep `derived.speed` — it's the raw time-series).
2. Stop writing `precomputedAnalytics.sessionMetrics` / `performance` / `accelerationSummary` / `highIntensity` in the Cloud Function.
3. Run a migration to strip those fields from existing `sensorData` docs.
4. `sensorData` becomes purely `{userID, uploadedAt, derived.speed, pages/*}` — no analytics at all.

This is the final state: a tiny queryable metrics collection + a big raw time-series collection, each optimal for its access pattern.

---

## What the phone is currently wasting

From the RN research:

1. **Computes acceleration internally but discards it.** `StatsEngine.ts` computes acceleration from the raw IMU data (needed for event counts and high-accel detection), but only writes `derived.speed` to Firestore. The Cloud Function then re-derives acceleration from speed at line ~1015 in `processAndSaveAnalytics()`. Pure waste.
   
   **Fix:** write `derived.accel: Array<{t, a}>` alongside `derived.speed` during upload. Also computes `highAccelEvents` and `highDecelEvents` and writes them as top-level fields on `sessionMetrics`.

2. **Discards rich `topSpeedMoments`, `timeline`, `phases` arrays.** These are computed in `computeStats()` but thrown away after the `sensorData` write. At least `phases` could be useful for the agent mode and calendar drilldowns — consider adding a `phaseStats` array field to `sessionMetrics`.

3. **Doesn't write `sessionType`.** The upload metadata has it but it's only hit via `sessionGroups/{groupId}` cross-reference today. Adding it directly to `sessionMetrics` eliminates a join.

4. **Doesn't write `teamId` or `playerId`.** Same — today those are derived via `sensorData.userID → players.where(userUid) → teamId`. Caching them on the metrics doc at upload time makes team/player queries a single Firestore read.

---

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| Phone write succeeds, batch doesn't commit (e.g. network drop) → drift between `sensorData` and `sessionMetrics` | Use a single Firestore `WriteBatch` for both docs. Either both commit or neither does. |
| Phone computes a metric inconsistently with the Cloud Function → website shows different numbers | Mirror every formula change in both places. CLAUDE.md's [[01 - Critical Preservation Rules#🧮 StatsEngine consistency across platforms|three-implementation rule]] already enforces this. Add a CI check that runs a fixed input through both StatsEngines and compares output. |
| Backfill script runs against a live DB and collides with concurrent Cloud Function writes | Backfill uses `set(..., {merge: true})` so it never clobbers Cloud-Function-written fields. Skip docs that already have `metricsLastSource === "cloud_function"`. |
| Field naming drift between RN, Cloud Function, and website (someone writes `maxSpeedKmh` in RN and `max_speed_kmh` in the CF) | Single source of truth TypeScript interface in `lib/types.ts`, imported by every platform that can. For the Swift/Obj-C that can't, a Swift enum file mirrors the field names with a runtime test. |
| `sessionMetrics` gets out of sync with `sensorData` over time (e.g. a doc gets deleted, its metrics don't) | Add a Cloud Function `onDocumentDeleted` trigger on `sensorData` that deletes the matching `sessionMetrics/{id}` — simple cascade. |
| Composite indexes are expensive if misused | Define only the indexes actually used by current queries. The Firebase Console will complain if a query needs one that doesn't exist, so we can add them reactively. |
| Phone is offline when the session ends → delayed write → sessionMetrics lag | Not a new risk. `SessionUploader.ts` already queues uploads; the batch just includes the new doc too. |

---

## Design answers (locked 2026-04-11)

All 4 open questions resolved by the architect:

1. **Fatigue names and direction — FIX NOW.** Rename to three unambiguous fields with distinct scales, and flip the sprint-speed metric so **higher = more fatigued** across all three (matching the Agent Mode tool label):
   - `sprintSpeedFatigueScore1to10` — was `fatigueScore`. 1 = sprints sustained, 10 = total sprint collapse. Stored as `11 - rawBucketScore` so the direction flips at write time.
   - `activityLoadIndex0to100` — was `performance.fatigueIndex`. 0 = standing still, 100 = max activity. Not renamed from the direction perspective (it's really a workload proxy, not fatigue), just given a clearer name.
   - `segmentedFatigueScore0to10` — was `segmentedFatigue.score`. **Flipped**: new formula = `10 - rawScore`. 0 = no decline, 10 = full collapse (matches the label everywhere).
2. **phaseStats — YES, include them.** StatsEngine already computes early/mid/late splits and throws them away. Include all three phases as flat top-level fields on `sessionMetrics`.
3. **Top-level collection — YES.** `sessionMetrics/{sessionId}`, not nested under `orgs/{orgId}`. Simpler cross-org admin queries, single place for the agent mode to scan.
4. **Phase stats flat — YES.** No subcollections. Each phase's metrics become `phase{Early|Mid|Late}DistanceM`, `phase{Early|Mid|Late}MaxSpeedKmh`, etc. on the main doc. Keeps queries simple.

The schema in the previous section already uses these names. Code below applies the direction flip at write time so every client that reads `sessionMetrics` sees consistent "higher = more fatigued" semantics without needing to know about the underlying formula direction.

## Additional phaseStats fields (from answer #2)

Add these to the `SessionMetricsDoc` schema. Values come from `StatsEngine.ts:computeStats().phases`, which currently produces an array `[early, mid, late]` of `PhaseStats` — we just flatten it:

```ts
// Phase breakdowns — EARLY = first third, MID = middle third, LATE = last third
// Every distance is in meters, every speed in km/h, every count absolute
phaseEarlyDistanceM: number
phaseEarlySprintDistanceM: number
phaseEarlyMaxSpeedKmh: number
phaseEarlyAvgSpeedKmh: number
phaseEarlySprintEvents: number
phaseEarlyDurationSec: number

phaseMidDistanceM: number
phaseMidSprintDistanceM: number
phaseMidMaxSpeedKmh: number
phaseMidAvgSpeedKmh: number
phaseMidSprintEvents: number
phaseMidDurationSec: number

phaseLateDistanceM: number
phaseLateSprintDistanceM: number
phaseLateMaxSpeedKmh: number
phaseLateAvgSpeedKmh: number
phaseLateSprintEvents: number
phaseLateDurationSec: number
```

18 extra fields × ~10 bytes each = ~200 bytes. Total doc size stays around ~1.7 KB. Enables queries like "show me players whose late-phase sprint events dropped > 50% below their early-phase count" — a direct in-DB fatigue query that doesn't need the raw time-series at all.

---

## Related

- [[Firestore Collection Audit 2026-04-11]] — the audit that triggered this migration
- [[Firebase Backend]] — collection reference
- [[Cresento React Native App]] — SessionUploader.ts, StatsEngine.ts, where Phase A lands
- [[Cresento Website#Agent Mode]] — Phase E tool rewrites
- [[StatsEngine Cross-Platform]] — the three-implementations rule this migration must respect
- [[01 - Critical Preservation Rules#🧮 StatsEngine consistency across platforms]] — the fatigue label bug callout
