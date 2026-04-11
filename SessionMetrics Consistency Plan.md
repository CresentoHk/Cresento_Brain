---
title: SessionMetrics Consistency Plan
type: design
tags:
  - shared
  - backend
  - frontend
  - firebase
  - migration
created: 2026-04-11
aliases:
  - Fields Consistency RN Website
  - Cross-Platform SessionMetrics Edit Plan
---

# SessionMetrics Consistency Plan

A file-by-file plan for keeping the new `sessionMetrics/{sessionId}` schema consistent across the React Native app and the Cresento website. This is the execution companion to [[SessionMetrics Migration Plan]].

> [!danger] Three implementations, one schema
> The same SessionMetrics doc will be written by **three independent code paths** (RN phone, Cloud Function backfill, website `lib/firestore.ts` helpers). Every field name, every unit, every direction flip must match exactly across all three. Drift between them means a coach's calendar on the web shows different numbers than their phone. This note exists so a future edit to any one of them can be cross-checked against every other call site.
>
> Pair with [[01 - Critical Preservation Rules#🧮 StatsEngine consistency across platforms]] — this extends the existing "three StatsEngines, same output" rule to cover the new SessionMetrics layer.

---

## Schema location — single source of truth

**The canonical schema lives in [[SessionMetrics Migration Plan#Schema (TypeScript)]].** Every code change in this plan must reference that schema, not an inlined copy.

Additionally, the schema is implemented in TypeScript at:

```
Cresento Website/Cresento.net/lib/types.ts
  → export interface SessionMetricsDoc { ... }
```

Both the website and the (future shared) RN build reference that interface. The one-shot backfill script `scripts/backfill-sessionMetrics.mjs` has its own inline schema since .mjs scripts can't import TS types — but every field name and unit in that script **must** match `lib/types.ts`.

---

## Phase A — React Native app writes `sessionMetrics` on upload

### Files to create

**`Cresento/src/src/utils/SessionMetricsBuilder.ts`** (NEW)

Pure function that takes the existing `SessionAnalytics` output from `StatsEngine.computeStats()` plus upload metadata, and produces a `SessionMetricsDoc`. No side effects. Imported from `SessionUploader.ts`.

```ts
// Pseudocode — mirrors scripts/backfill-sessionMetrics.mjs buildSessionMetricsDoc()
export function buildSessionMetricsDoc(
  sessionId: string,
  analytics: SessionAnalytics,    // from StatsEngine.computeStats()
  metadata: {
    userId: string
    orgId: string | null
    playerId: string | null
    teamId: string | null
    sessionType: "game" | "training" | "session"
    uploadedAt: Timestamp
    sessionStartTime: Timestamp
    sessionEndTime: Timestamp
  }
): SessionMetricsDoc { ... }
```

Responsibilities:
- Flatten `analytics.stats` → top-level fields
- Flatten `analytics.phases[]` → 18 phase fields (`phaseEarlyDistanceM`, ...)
- **Apply direction flip for `sprintSpeedFatigueScore1to10`** — `11 - analytics.stats.fatigueScore` with 1–10 clamp
- Compute per-minute rates (`distancePerMinM`, `sprintEventsPerMin`, etc.) using `analytics.stats.minutesPlayed` as the duration
- Set cross-session fields to null (`speedRegression*`, `acwr*`, `segmentedFatigueScore0to10`) — these are filled in later by the Cloud Function
- Set `metricsLastSource: "phone"`
- Stamp `metricsComputedAt: Timestamp.now()`

### Files to edit

**`Cresento/src/src/utils/SessionUploader.ts`** (MODIFY — sensitive)

> [!warning] This file is on the [[01 - Critical Preservation Rules|preservation list]]
> It owns the 700 KB page cap logic. Do NOT touch the paging code. The edit below is strictly additive — a second doc gets written in the SAME batch as the existing `sensorData` write.

Location of the edit: inside `uploadSession()`, after `computeStats()` succeeds and before the `sensorData` doc write.

```ts
// AFTER analytics computation succeeds (around line 168 today):
const analytics = computeStats(csvText)
if (analytics) {
  // BUILD the new sessionMetrics doc alongside the existing sensorData doc
  const metricsDoc = buildSessionMetricsDoc(sessionId, analytics, {
    userId: uid,
    orgId: metadata.orgId ?? null,
    playerId: metadata.playerId ?? null,
    teamId: metadata.teamId ?? null,
    sessionType: metadata.sessionType ?? "session",
    uploadedAt: serverTimestamp(),
    sessionStartTime,
    sessionEndTime,
  })

  // Write BOTH docs in a single batch for atomicity
  const batch = writeBatch(db)
  batch.set(doc(db, "sensorData", sessionId), sensorDataPayload)
  batch.set(doc(db, "sessionMetrics", sessionId), metricsDoc)
  await batch.commit()
} else {
  // Fallback: still write sensorData even if analytics failed
  await setDoc(doc(db, "sensorData", sessionId), sensorDataPayload)
}
```

Rule: **never** write `sessionMetrics` without also writing `sensorData` — use the batch, always. Drift between the two means the website will show ghost sessionMetrics docs for sessions whose raw data never made it.

**Paging caveat:** when the CSV text is too big (> 700 KB → gets paged into `pages/0`, `pages/1`, ... subcollections), the current code uses a slightly different write path (line 215–251). The `sessionMetrics` write should still happen in the same top-level batch, NOT inside the per-page writes. One `sessionMetrics` doc per session regardless of paging.

### Files to search for follow-on edits in RN

- `Cresento/src/src/contexts/BleContext.tsx` — if it has its own upload path that bypasses `SessionUploader`, audit it. I recall from earlier research that there's a recovery path in `HomeScreen.tsx` that writes to `users/{uid}/sessions` — that recovery path needs the same `sessionMetrics` write OR we explicitly document that recovery uploads skip `sessionMetrics` (users have to re-upload through the normal flow).
- `Cresento/src/src/utils/StatsEngine.ts` — read-only, do not edit. The builder consumes its output.
- `Cresento/src/src/pages/HomeScreen.tsx` — if it calls `SessionUploader.uploadSession()`, nothing to do. If it has its own direct Firestore writes for sessionData, they need the parallel batch update.

### Verification for Phase A

1. After the RN app ships with the new write, upload a test session from a player phone.
2. Check that both `sensorData/{id}` and `sessionMetrics/{id}` exist with the same ID.
3. Fields to verify match between the two:
   - `sessionMetrics.totalDistanceM` === `sensorData.derived.stats.distanceStepsMeters` (rounded to int)
   - `sessionMetrics.maxSpeedKmh` === `sensorData.derived.stats.maxSpeedKmh` (rounded to 2dp)
   - `sessionMetrics.sprintSpeedFatigueScore1to10` === `11 - sensorData.derived.stats.fatigueScore` (direction flipped)
4. Run a golden-input test: feed a known CSV into `StatsEngine.computeStats()` + `buildSessionMetricsDoc()` and compare the output to a checked-in fixture JSON. Any field drift breaks the test.

---

## Phase B — Cloud Function updates `sessionMetrics` with cross-session fields

### Files to edit

**`Cresento Website/Cresento.net/functions/src/index.ts`** (MODIFY)

The existing `processAndSaveAnalytics()` function (lines 970–1109) currently writes `precomputedAnalytics` to `sensorData/{id}` and writes heavy analytics (acceleration, segmentedFatigue, distanceBreakdown, eventCounts) to the `sensorData/{id}/additional_data/analytics` subcollection.

Add a parallel update to `sessionMetrics/{id}`. Specifically:

```ts
// AFTER the existing processAndSaveAnalytics work:
const sessionMetricsRef = db.collection("sessionMetrics").doc(docId)
const sessionMetricsUpdate = {
  // Regression fields (from precomputedAnalytics.regression)
  speedRegressionSlope: round(regression.slope, 4),
  speedRegressionIntercept: round(regression.intercept, 4),
  speedRegressionRSquared: round(regression.rSquared, 4),
  speedDeclinePct: round(regression.speedDeclinePct, 2),
  speedLoss: round(regression.speedLoss, 3),
  staminaDecay: round(performance.staminaDecay, 4),
  consistencyScore: round(performance.consistencyScore, 2),

  // The zone-weighted activity index (was "performance.fatigueIndex")
  activityLoadIndex0to100: round(performance.fatigueIndex, 2),

  // Segmented fatigue — FLIPPED so higher = more fatigued
  segmentedFatigueScore0to10: segmentedFatigue
    ? round(10 - segmentedFatigue.score, 2)
    : null,
  segmentedFatigueConfidence: segmentedFatigue?.confidence ?? null,

  // Provenance
  metricsComputedAt: FieldValue.serverTimestamp(),
  metricsLastSource: "cloud_function",
}
await sessionMetricsRef.update(sessionMetricsUpdate)
```

**Use `update()` not `set()`** — the phone already wrote the phone-computed fields in Phase A, and `set()` without merge would wipe them. If you're paranoid, wrap in a transaction that reads the current doc first and only writes fields that aren't already phone-written.

Similarly update the daily ACWR function (`calculateDailyACWR`, line 1434) to write ACWR fields to `sessionMetrics/{latestSessionId}` with the same `update()` pattern:

```ts
await db.collection("sessionMetrics").doc(latestSessionId).update({
  acwrValue: round(acwrResult.value, 3),
  acwrInjuryRisk: acwrResult.injuryRisk,
  acwrAcuteLoad: round(acwrResult.acuteLoad, 2),
  acwrChronicLoad: round(acwrResult.chronicLoad, 2),
  acwrAcuteDays: acwrResult.acuteDays,
  acwrChronicDays: acwrResult.chronicDays,
  acwrComputedAt: FieldValue.serverTimestamp(),
})
```

### Cloud Function code to delete (Phase B cleanup)

Once Phase A is writing phone-computable fields to `sessionMetrics` directly, the following blocks in `functions/src/index.ts` can be stripped from `processAndSaveAnalytics()`:

- `calculateSessionMetrics()` call (computes cadence / workRate / sprint frequency — the phone now does this)
- `computeDistanceBreakdown()` call (HSR / high-accel distances — phone-computable)
- `computeEventCounts()` call (phone-computable)
- `accelerationSummary` block (phone discards acceleration today; we should fix THAT and have the phone write it instead)

Estimated Cloud Function runtime reduction: ~60% based on the research agent's earlier findings.

### Cross-session fields that MUST stay in the Cloud Function

These require historical data the phone doesn't have:
- `regression.*` — requires time-over-time trend analysis
- `segmentedFatigueScore0to10` — requires windowed comparison (could technically move to phone if we port the 8-ratio algorithm from `analytics-utils.ts`, but not today)
- `acwr.*` — requires 7-day + 28-day rolling aggregation across sessions

---

## Phase C — already done (one-shot backfill)

`scripts/backfill-sessionMetrics.mjs` ran 2026-04-11. It populated `sessionMetrics/{id}` for every existing `sensorData/{id}` doc in the DB. Idempotent — running it again skips docs that already exist unless `--force` is passed.

The backfill uses its own inline schema implementation (.mjs can't import the TS interface). Any field addition to `SessionMetricsDoc` in `lib/types.ts` that's supposed to be backfillable for existing sessions needs to also be added to `scripts/backfill-sessionMetrics.mjs` — otherwise existing sessions will have the new field set to null forever until re-uploaded.

---

## Phase D — Website reads `sessionMetrics` instead of `sensorData`

### Files to create

**`Cresento Website/Cresento.net/lib/session-metrics.ts`** (NEW)

Thin wrapper module with typed getters for the new collection. Mirrors the existing `lib/firestore.ts` pattern.

```ts
import type { SessionMetricsDoc } from "./types"

export async function getSessionMetrics(sessionId: string): Promise<SessionMetricsDoc | null>

export async function getSessionMetricsByOrg(
  orgId: string,
  opts?: {
    dateFrom?: Date
    dateTo?: Date
    sessionType?: "game" | "training" | "session"
    limit?: number
  }
): Promise<SessionMetricsDoc[]>

export async function getSessionMetricsByPlayer(
  userId: string,
  opts?: { limit?: number; rangeDays?: number }
): Promise<SessionMetricsDoc[]>

export async function getSessionMetricsByTeam(
  teamId: string,
  opts?: { limit?: number; rangeDays?: number }
): Promise<SessionMetricsDoc[]>

/** Fast calendar-list helper — replaces full sensorData scan */
export async function listSessionMetricsForCalendar(
  orgId: string,
  monthKeys: string[]
): Promise<SessionMetricsDoc[]>
```

### Files to edit

**`Cresento Website/Cresento.net/lib/types.ts`** (ADD export)

Add the `SessionMetricsDoc` interface from the migration plan. This becomes the shared type used by both the website code and (via duck-typing) the RN SessionMetricsBuilder.

**`Cresento Website/Cresento.net/lib/firestore.ts`** (MODIFY)

This is the ~4600-line wrapper. The following functions have existing callers that ONLY need the aggregate metrics (not raw time-series). Swap them to read from `sessionMetrics` instead of `sensorData`:

- `getSensorDataSummariesByOrg` → rename to `getSessionMetricsByOrg` (OR keep the old name, change the body)
- `listSessionGroupsByOrg` — unchanged (it reads session groups, not sensor data)
- Any function that reads `sensorData.derived.stats` without also reading `derived.speed[]` — those are candidates for direct `sessionMetrics` reads

Functions that MUST stay on `sensorData`:
- Anything that reads `derived.speed[]` — chart rendering, raw data endpoints
- The session-detail page (needs the full raw series for the speed chart)
- `getFullSensorDataCached` — used by the agent mode's `analyze_with_code` sandbox

**`Cresento Website/Cresento.net/components/dashboard/sessions-calendar.tsx`** (MODIFY — the 70 KB critical file)

This is the biggest per-user win. The calendar today loads `sensorData` summaries to display per-session cards. After the migration, it should:

1. Load `listSessionMetricsForCalendar(orgId, monthKeys)` — one small query per month
2. Display per-session cards directly from the flat fields (`maxSpeedKmh`, `totalDistanceM`, etc.)
3. ONLY fall through to `getFullSensorDataCached(sessionId)` when the user clicks a session to see the detailed chart view

Expected speedup: calendar page-load time drops from ~4 seconds to ~120 ms for a typical team.

> [!warning] Sessions calendar is sacred
> Per [[01 - Critical Preservation Rules#🗓️ Calendar view session loader]], don't refactor anything in this file beyond the data-source swap. Keep the rendering, grouping, trim logic, everything. Only change "where the metrics come from."

### Other components to audit

| Component | Action |
|---|---|
| `components/analytics/class-comparison.tsx` | Already reads `ClassAverages/football` — unaffected by this migration. But we ALSO want to move `ClassAverages` to `config/benchmarks/football` (see audit note). |
| `components/analytics/trends-comparison.tsx` | Same |
| `components/coach/coach-console.tsx` | Replace `sensorData` summary reads with `sessionMetrics` reads |
| `components/dashboard/*-card.tsx` | Any card showing aggregate metrics per session — migrate |

---

## Phase E — Agent Mode tools

### Files to edit

**`Cresento Website/Cresento.net/lib/agent/tools.ts`** (MODIFY — big file)

The agent mode currently has a `SUMMARY_STAT_MAP` extractor (tools.ts:628) that pulls specific metric fields out of nested `sensorData.derived.stats` and `precomputedAnalytics` objects. After the migration, these are all flat fields on `sessionMetrics` — the extractor table becomes unnecessary.

Tool-by-tool migration:

| Tool | Change |
|---|---|
| `get_team_roster` | No change (reads `players/`) |
| `get_teams` | No change |
| `list_player_sessions` | Change backend to query `sessionMetrics` — add fields like `maxSpeedKmh` inline so the list becomes more useful |
| `get_recent_sessions` | No change (reads `sessionGroups/`) |
| `browse_calendar` | Change to read pre-aggregated `sessionMetrics` instead of scanning raw sensor data |
| `query_team_overview` | **BIG win**. Replace the `getSensorDataSummariesByOrg` + in-memory filter with a direct Firestore query like `sessionMetrics.where("orgId","==",orgId).where("uploadedAt",">=",cutoff).orderBy("<metric>","desc").limit(N)`. Drops from ~2.5 MB per query to ~75 KB. |
| `load_game_data` | Keep reading `sessionGroups` + player lookups. No change. |
| `query_game_metrics` | Replace `getFullSensorDataCached(sessionId)` reads with `getSessionMetrics(sessionId)` reads. Drops from ~5 MB/player to ~1.5 KB/player. |
| `query_session_metrics` | Same — swap to `sessionMetrics`. |
| `query_metric_summary` | **BIG win**. Replace in-memory trend computation with Firestore `orderBy` queries. The trend analysis logic in `lib/analytics/trends.ts` still applies — just pulls from smaller docs. |
| `query_player_insights` | No change (reads `getPlayerProgress()`) |
| `compare_players` | Swap to parallel `getSessionMetricsByPlayer` calls |
| `get_session_timeseries` | **UNCHANGED** — this tool is the reason `sensorData` still exists. Keep reading raw. |
| `analyze_with_code` | **UNCHANGED** — the sandbox needs full 20 Hz raw data from `sensorData`. |
| `browse_charts` | No change |
| `render_chart` | Keep reading `sensorData` for the raw speed arrays needed to render charts |

**`Cresento Website/Cresento.net/lib/agent/metric-docs.ts`** (MODIFY)

Update the metric docs to match the new flat field names:

- `fatigueIndex` entry → split into two distinct entries: `sprintSpeedFatigueScore1to10` and `activityLoadIndex0to100`. Delete the "label bug" warnings — the direction is now correct everywhere.
- `segmentedFatigueScore` entry → rename `key` to `segmentedFatigueScore0to10`. Update the formula block to reflect the 10 - rawScore flip.
- Add 18 new metric docs for the phase stat fields (one per phase × 6 metrics). These can be short since they all share the same "phase = early/mid/late third of session" explanation.

**`Cresento Website/Cresento.net/lib/agent/config.ts`** (MODIFY)

The system prompt mentions fatigue multiple times. Swap references like "fatigueScore" to the new explicit names. Also update the "Recipes" section: e.g. "for 'who's my worst player' queries, start with `sessionMetrics.where(orgId).orderBy(workRateMPerMin, 'asc').limit(5)` — then only load `sensorData` for deeper drilldown."

---

## Phase F — RN app reads `sessionMetrics`

If the RN app displays per-session lists (leaderboards, player progress pages, calendar views), migrate those read paths to `sessionMetrics` too. Same speed gains apply to the phone.

Files most likely to be affected:
- `Cresento/src/src/pages/PlayerPage.tsx` (if it exists)
- `Cresento/src/src/pages/CalendarScreen.tsx`
- Any player-progress views
- Any coach-dashboard views inside the RN app

Audit these for direct `sensorData` reads and migrate them to new helpers:

**`Cresento/src/src/utils/SessionMetricsReader.ts`** (NEW)

Mirror of the website's `lib/session-metrics.ts` — typed getters for the RN app:

```ts
export async function getSessionMetricsByPlayer(userId: string): Promise<SessionMetricsDoc[]>
export async function getSessionMetricsByTeam(teamId: string): Promise<SessionMetricsDoc[]>
// ...
```

Uses the RN Firebase SDK (not Admin SDK, not the website's client SDK) so it'll have its own imports. But the return TYPE is the same `SessionMetricsDoc` — share via a common types file if possible, or duplicate with a test ensuring shape equality.

---

## Phase G — Deprecate `derived.stats` and `precomputedAnalytics` subsets

After Phases A–F are deployed and verified, clean up the legacy paths:

1. **Stop writing `derived.stats`** in `SessionUploader.ts`. Keep `derived.speed` — it's the raw time-series, still needed.
2. **Stop writing `precomputedAnalytics.performance`, `.sessionMetrics`, `.accelerationSummary`, `.highIntensity`** in the Cloud Function. Those are all now on `sessionMetrics`.
3. **Keep** `precomputedAnalytics.regression` (if the Cloud Function still uses it internally) and `precomputedAnalytics.acwr` (backwards compat) OR move them entirely to `sessionMetrics` and delete from the sensorData doc.
4. **Delete** the `sensorData/{id}/additional_data/analytics` subcollection entirely. Its contents (segmentedFatigue, distanceBreakdown, eventCounts) are all on `sessionMetrics` now.
5. **Run a one-time migration** that strips the deprecated fields from every existing `sensorData` doc (using `FieldValue.delete()` in a batch loop). This is where the actual SPACE savings materialize.

Expected post-Phase-G state of `sensorData/{id}`:
```
{
  id, userID, uploadedAt, location, sessionNumber,
  sessionStartTime, sessionEndTime, sessionDuration,
  dataType, dataSize,
  derived: {
    speed: Array<{t, v}>   // raw 20 Hz time-series, nothing else
  },
  pageCount?, pagesReady?,  // still paged for CSV text
  // + pages/* subcol for the raw CSV
  // NO more derived.stats
  // NO more precomputedAnalytics
  // NO more additional_data
}
```

Shrinks a typical session doc from ~5 MB to ~1–2 MB. Combined with Phase G migration, the whole `sensorData` collection's storage footprint drops 40–60%.

---

## Consistency verification checklist

After each phase, run this checklist to catch schema drift:

- [ ] **Phase A golden test** — feed a fixed CSV through `StatsEngine.computeStats()` + `buildSessionMetricsDoc()`, compare output JSON to a checked-in fixture file. CI test blocks merges on any field drift.
- [ ] **Phase B idempotency test** — run the Cloud Function twice on the same sensorData doc, confirm the final `sessionMetrics` doc is identical (no duplicated counters, no overwritten phone-written fields).
- [ ] **Phase C correctness** — spot-check 20 random `sessionMetrics` docs after backfill. Fields should match a manual computation from the source `sensorData.derived.stats`. The backfill script prints 2 samples by default in verbose mode.
- [ ] **Phase D cross-check** — load a session on the website calendar AND via the new `getSessionMetrics()` helper. Every displayed number should match.
- [ ] **Phase E agent test** — run a fixed query (e.g. "who's my top sprinter in the last 30 days") before and after Phase E. Same player, same number, ~10x faster.
- [ ] **Phase F RN test** — same session shown in the RN player progress view should match the website value.
- [ ] **Phase G post-deploy** — verify `sensorData` docs are smaller, chart rendering still works, agent sandbox still has raw data.

---

## Who owns what

| Concern | Owner | File |
|---|---|---|
| Schema source-of-truth | — | `lib/types.ts` → `SessionMetricsDoc` |
| Phone-computed fields | RN | `Cresento/src/src/utils/SessionMetricsBuilder.ts` |
| Cloud-computed fields | Cloud Function | `functions/src/index.ts` |
| Backfill for existing | One-shot | `scripts/backfill-sessionMetrics.mjs` |
| Website reads | Web | `lib/session-metrics.ts` + `lib/firestore.ts` |
| Agent reads | Web | `lib/agent/tools.ts` + `lib/agent/metric-docs.ts` |
| RN reads | RN | `Cresento/src/src/utils/SessionMetricsReader.ts` |

---

## Related

- [[SessionMetrics Migration Plan]] — full schema, design answers, phase overview
- [[Firestore Collection Audit 2026-04-11]] — what got cleaned up to make room for this
- [[Firebase Backend]] — collection reference
- [[Cresento React Native App]] — where Phase A + F land
- [[Cresento Website#Agent Mode]] — where Phase E lands
- [[StatsEngine Cross-Platform]] — the existing cross-platform consistency rule
- [[01 - Critical Preservation Rules#🧮 StatsEngine consistency across platforms]] — the fatigue label bug callout (resolved by this migration)
