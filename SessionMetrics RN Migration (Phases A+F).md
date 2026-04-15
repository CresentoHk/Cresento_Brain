---
title: SessionMetrics RN Migration (Phases A+F)
type: design
tags:
  - project/rn-app
  - shared
  - backend
  - performance
  - critical
created: 2026-04-14
status: phase-a-done
---

## SessionMetrics RN Migration — Phases A + F

This note documents what needs to change in the React Native app (`Cresento/`) for the SessionMetrics migration. It covers two phases:

- **Phase A (phone writes)** — The phone writes a lightweight `sessionMetrics/{id}` doc alongside the heavy `sensorData/{id}` doc at upload time.
- **Phase F (RN reads)** — The app reads session listings and pre-computed stats from `sessionMetrics` instead of downloading multi-MB `sensorData` docs.

> [!info] Phase A is ALREADY DONE
> `SessionUploader.ts` already writes `sessionMetrics/{id}` atomically alongside `sensorData/{id}`. `SessionMetricsBuilder.ts` produces the flat doc. No further work needed.

---

## Phase A — Phone Writes (DONE)

### What was built

Two files implement the phone-side write:

**`src/src/utils/SessionMetricsBuilder.ts`** — Pure builder that converts `SessionAnalytics` from `StatsEngine.computeStats()` into a flat `SessionMetricsDoc`. Matches the exact same schema as the backfill script and Cloud Function, so every client sees identical field names and units.

**`src/src/utils/SessionUploader.ts`** — Modified to write both collections in the same batch:
- Single-doc path (lines ~295–341): Batch-writes `sensorData/{id}` + `sessionMetrics/{id}` atomically
- Paged path (lines ~344–431): Writes `sensorData/{id}` + pages subcollection, then writes `sessionMetrics/{id}` in separate batch
- Identity resolution via `resolvePlayerContext(uid)` queries `players/` collection to get `orgId`, `playerId`, `teamId`

### Key design rules (must preserve)

- Every metric field is flat, top-level — no nested sub-objects
- Missing data is `null`, not `0`
- Fatigue direction is FLIPPED so higher = more fatigued (`sprintSpeedFatigueScore1to10 = 11 - rawFatigueScore`)
- Cross-session fields (`acwr*`, `speedRegression*`, `segmentedFatigue*`) are left `null` on the phone — the Cloud Function fills them later via `update()`
- Schema must stay identical to: `Cresento Website/Cresento.net/lib/types.ts → SessionMetricsDoc` and `scripts/backfill-sessionMetrics.mjs`

---

## Phase F — RN Reads (TODO)

### What "RN reads" means

Today, when the React Native app needs to display a list of sessions (player history, coach viewing a player, leaderboards), it queries the `sensorData` collection. Each `sensorData` doc is 50 KB–5 MB because it contains raw CSV time-series data, speed arrays, and paged sub-documents.

**The problem:** The app only needs a handful of summary numbers (date, duration, location, top speed, distance, sprint count) to render a session list card. But it downloads the entire multi-MB doc to get them.

**The fix:** Read from `sessionMetrics` instead. Each `sessionMetrics` doc is ~1.5 KB and contains all the summary metrics as flat top-level fields. Same doc ID as `sensorData`, so joining back to the full data for drill-down is trivial.

**Phase F does NOT change:**
- Session detail view (still needs raw CSV from `sensorData` to render time-series charts)
- Session upload (Phase A, already done)
- Session deletion (still needs to delete both `sensorData` and `sessionMetrics`)

### Files that need changes

#### 1. `src/src/pages/Statistics.tsx` — Player Session History

**Current code (lines 84–89, 146–152):**
```typescript
const q = query(
  collection(db, 'sensorData'),           // <── CHANGE THIS
  where('userID', '==', currUser.uid),
  orderBy('uploadedAt', 'desc'),
  limit(SESSION_PAGE_SIZE),
);
```

Extracts these fields from each doc (lines 98–114):
- `uploadedAt`, `sessionDuration`, `rawCSV`, `pageCount`, `pagesReady`, `sessionStartTime`, `dataType`, `dataSize`, `location`

**What to change:**
1. Change `collection(db, 'sensorData')` → `collection(db, 'sessionMetrics')`
2. Change `where('userID', ...)` → `where('userId', ...)` (note: `sessionMetrics` uses `userId` not `userID` — the field name was fixed during the migration)
3. Update the field extraction to use sessionMetrics field names:
   - `data.sessionDuration` → `data.durationMin` (already in minutes)
   - `data.rawCSV` → not available in sessionMetrics (not needed for list view)
   - `data.sessionStartTime` → `data.sessionStartTime` (same name)
   - `data.dataType` → not available (not needed for list view)
   - `data.dataSize` → not available (not needed for list view)
   - `data.location` → not available in sessionMetrics yet (see "Schema gap" below)
4. **Also add pre-computed stats** to the session object — sessionMetrics has `maxSpeedKmh`, `totalDistanceM`, `sprintEvents`, `durationMin` etc. as flat fields, so the list view can show summary stats without navigating into detail
5. Apply the same change to the `loadMoreSessions` pagination query (lines 146–152)
6. Keep navigating to `SessionDetail` with the `sensorData` doc ID (same ID as `sessionMetrics`)

**Important:** `rawCSV` and `pageCount`/`pagesReady` are NOT in `sessionMetrics`. These are only needed when opening the detail view. The Statistics page passes them via route params to SessionDetail. After this migration:
- The list view no longer carries `rawCSV` (saves memory)
- SessionDetail will need to fetch `rawCSV` from `sensorData/{id}` on demand when opened (currently it receives it via route params)

#### 2. `src/src/hooks/useCoachPlayerViewModel.ts` — Coach Views Player

**Current code (lines 77–82):**
```typescript
const q = query(
  collection(db, 'sensorData'),           // <── CHANGE THIS
  where('userID', '==', playerUid),
  orderBy('uploadedAt', 'desc'),
  limit(50),
);
```

Extracts (lines 87–118):
- `uploadedAt`, `location`, `sessionNumber`, `rawCSV`, `pageCount`, `pagesReady`
- `derived.stats` → `session.stats`
- `derived.speed` → `session.speedSeries`

**What to change:**
1. Change `collection(db, 'sensorData')` → `collection(db, 'sessionMetrics')`
2. Change `where('userID', ...)` → `where('userId', ...)`
3. Drop `rawCSV`, `pageCount`, `pagesReady` from the session object (not in sessionMetrics, not needed for list)
4. Replace `derived.stats` extraction (lines 106–114) with direct flat field reads:
   ```typescript
   // OLD: derived.stats.maxSpeedKmh
   // NEW: data.maxSpeedKmh (flat top-level field)
   session.stats = {
     maxSpeedKmh: data.maxSpeedKmh,
     totalDistanceM: data.totalDistanceM,
     sprintEvents: data.sprintEvents,
     durationMin: data.durationMin,
     // ... map whichever fields the CoachSession type needs
   };
   ```
5. Drop `derived.speed` — speed time-series is NOT in sessionMetrics (only in sensorData). If the coach list view renders sparkline charts, those will need to be fetched on demand from sensorData when tapped.

#### 3. `src/src/pages/SessionDetail.tsx` — Session Detail (PARTIAL CHANGE)

**Current behavior:** Receives the full session data via route params (including `rawCSV`). Calls `computeStats(csvText)` locally to render analytics.

**What changes:**
1. Since Statistics.tsx will no longer pass `rawCSV` via route params, SessionDetail needs to fetch it on demand:
   ```typescript
   // On mount, load raw data from sensorData if not already cached
   const sessionRef = doc(db, 'sensorData', sessionData.id);
   const snap = await getDoc(sessionRef);
   const rawCSV = snap.data()?.rawCSV;
   ```
2. For paged sessions, the existing page-loading logic stays the same
3. **Delete must also delete sessionMetrics:** When deleting a session (lines 324–349), add:
   ```typescript
   // Also delete the sessionMetrics doc
   const metricsRef = doc(db, 'sessionMetrics', sessionId);
   await deleteDoc(metricsRef);
   ```

#### 4. `src/src/pages/Leaderboard.tsx` — Leaderboard (STUB)

Currently just a placeholder. When implemented, it should query `sessionMetrics` directly — this is exactly the use case sessionMetrics was designed for (e.g., "top 10 players by maxSpeedKmh this week").

### Schema gap: `location` field

The `sessionMetrics` schema does NOT currently include a `location` field. The session list views display location. Two options:

**Option 1 (recommended):** Add `location: string | null` to `SessionMetricsDoc` schema in all three locations:
- `SessionMetricsBuilder.ts` (phone write)
- `lib/types.ts → SessionMetricsDoc` (website types)
- `scripts/backfill-sessionMetrics.mjs` (backfill)

Then re-run the backfill for existing sessions. This is the cleanest fix.

**Option 2:** Keep reading `location` from a secondary `sensorData` query. Ugly, defeats the purpose.

### Schema gap: `sessionNumber` field

The coach view also reads `sessionNumber` from `sensorData`. This is NOT in `sessionMetrics`. Same options as location — add it to the schema or drop it from the list view.

---

## Firestore index requirements

The RN app's queries will need composite indexes on `sessionMetrics`:

```
sessionMetrics: userId ASC, uploadedAt DESC
```

This index may already exist from the website migration. Check `firestore.indexes.json` before adding duplicates.

---

## Testing checklist

- [ ] Upload a new session from the phone → verify `sessionMetrics/{id}` doc appears with correct fields
- [ ] Open Statistics tab → sessions load from `sessionMetrics` (check Firestore reads in logs)
- [ ] Tap a session → SessionDetail loads raw data from `sensorData` on demand
- [ ] Delete a session → both `sensorData/{id}` and `sessionMetrics/{id}` are removed
- [ ] Coach views a player → session list loads from `sessionMetrics`
- [ ] Coach taps a session → detail view loads raw data correctly
- [ ] Pagination (load more) works on Statistics page
- [ ] Real-time listener (`onSnapshot`) fires correctly on `sessionMetrics`

---

## Related

- [[SessionMetrics Migration Plan]] — master 7-phase plan
- [[SessionMetrics Consistency Plan]] — how phone / cloud / website produce matching values
- [[Cresento React Native App]] — RN app overview
- [[Firebase Backend]] — collection schema
- [[Data Pipeline]] — upload flow
