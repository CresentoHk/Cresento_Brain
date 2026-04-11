---
title: Firestore Collection Audit 2026-04-11
type: audit
tags:
  - backend
  - firebase
  - audit
  - shared
created: 2026-04-11
aliases:
  - Firebase Cleanup 2026-04-11
  - Collection Audit April 2026
---

# Firestore Collection Audit — 2026-04-11

A point-in-time snapshot of every Firestore collection in `cresento-8b603`, what each one is used for, which ones are dead, and the cleanup plan.

> [!info] Why this note exists
> The Firebase console showed **30 top-level collections** — more than the codebase actually uses. This audit maps every collection to its code callers (or lack thereof), counts the docs, and classifies each one as keep / delete / relocate / needs-TTL. Cleanup work tracked against this note.

---

## Tooling

Two new scripts live in `Cresento Website/Cresento.net/scripts/`:

- **`backup-firestore.mjs`** — Admin-SDK walker that dumps every collection (or a targeted subset) to local JSON. Recursively descends into subcollections. Streams in 500-doc batches. Handles Timestamp/GeoPoint/DocumentReference/Buffer serialization. Supports `--collections`, `--skip`, `--skip-subcollections`, `--dry-run`.
- **`restore-firestore.mjs`** — Inverse walker. Requires `--backup <path>` and `--yes` to actually write. Rebuilds exact Timestamp etc. values from the JSON markers. Supports `--collections` and `--merge`.

Both authenticate via `GOOGLE_APPLICATION_CREDENTIALS` pointing at a service account JSON (preferred — one env var, no credential paste into chat) or the three `FIREBASE_ADMIN_*` env vars.

> [!warning] Known bug fixed 2026-04-11
> Initial version of `backup-firestore.mjs` failed on `sessionGroupDismissals` with ENOENT because dismissal doc IDs are 280+ character concatenations of multiple session IDs — Windows MAX_PATH is 260. Fixed by capping filenames at 180 chars and appending a SHA-1 suffix for uniqueness when truncated. The full original id is still preserved in the JSON payload's `path` field, so restore still works.

`firestore-backup/` is gitignored — backups should never land in the repo.

---

## Full inventory with doc counts

Counts captured 2026-04-11 via `node scripts/backup-firestore.mjs --dry-run --skip-subcollections sensorData`.

### Healthy active collections (keep)

| Collection | Docs | Written by | Read by |
|---|---:|---|---|
| `sensorData` | 1,425 | iOS BLE upload, Cloud Function mergers | website, iOS, RN, agent mode |
| `users` | 135 | website signup, iOS signup | everywhere |
| `rawdata` | 82 | iOS BLE upload (metadata only, CSV in Storage) | *nothing on website* |
| `players` | 70 | website teams page, invites, `ensurePlayerRecord` | everywhere |
| `padOwners` | 32 | iOS + RN BLE pairing | iOS + RN coach device manager |
| `sessionGroups` | 29 | website `createSessionGroup`, auto-classifier | website, agent mode |
| `orgs` | 21 | website onboarding/teams | everywhere |
| `team_monthly_sessions` | 20 | Cloud Function `onSensorDataWrite` | website coach console, agent mode |
| `teams` | 12 | website teams page | everywhere |
| `sessionGroupDismissals` | 12 | website coach page | website coach page |
| `coachConsoleCache` | 8 | Cloud Function | website coach page |
| `liveStatus` | 7 | iOS BLE recording | iOS Coach view |
| `scheduledActivities` | 2 | website coach page | website coach page |
| `config` | 1 | manual via seed scripts | server-side only (Agent Mode key pool) |
| `agentUsage` | 0 | `lib/agent/usage-log.ts` (new in V2) | server-side only |

> [!info] Why `agentUsage` is empty
> As of this audit, no Agent Mode V3 request has successfully landed. Either (a) the deploy hasn't been exercised, or (b) the auth guard is silently rejecting every call. Worth monitoring — if no `agentUsage/{uid}` docs appear after real usage, that's a separate bug to investigate.

### Stale accumulators (need TTL, not deletion)

| Collection | Docs | Problem |
|---|---:|---|
| `teamInvites` | **128** | Invites never expire — should auto-delete after ~7 days |
| `emailCodes` | **116** | Email verification codes — should auto-delete after ~24h |

Fix: add a Firestore TTL policy in the Firebase Console. Zero code changes.

### Ghost collections — DELETE (confirmed dead)

All have 1-3 docs and zero live code references. JSON dumps backed up to `firestore-backup/2026-04-11T03-42-52-129/`.

| Collection | Docs | Evidence |
|---|---:|---|
| `IdealMetrics` | 1 | Twin of `ClassAverages/football` with slightly different values + `notes` fields. Abandoned earlier version of the benchmark table. Not read anywhere. |
| `PadClaims` | 1 | Old pad claim record from 2025-07-25. Renamed to `padOwners` long ago. One stale doc with the same UUID as the live `padOwners` record. |
| `pads` | 1 | Single doc at `pads/78A07027-...` — a pad record with the same device UUID (uppercase) as the PadClaims doc. Zero references to `collection(db, "pads")`. `pads` is a FIELD on user docs in the iOS code (`data["pads"] as? [String:String]`), not a collection. |
| `Team` (capital T) | 1 | Single doc `Team/V8GJNW` with `members[]` array structure — completely different schema from the live `teams/` collection which uses `coachUids[]`. Leftover from an older schema version. |
| `TrainingPrograms` | 3 | Two are obvious test data (`sample_player_1`, `test_player_123`), one is a "Standard Training Program" template from 2025-06-18 that was never wired up to any UI. |
| `userProgress` | 9 | All 9 docs have the identical template (`currentSession: "session1"`, `completedSessions: []`, `status: "active"`) with `startDate` timestamps all within the same minute on 2025-06-18. Seed data from one test run, never used since. |

### ~~Ghost collections — NEEDS CLIENT CHECK BEFORE DELETE~~ → CLEARED FOR DELETION

| Collection | Docs | Notes |
|---|---:|---|
| `exercises` | 3 | Two well-formed exercise definitions (`agility_drills`, `sprint_training`) with name/description/phase/category/equipment, plus one empty placeholder doc. **Verified 2026-04-11:** zero references in the RN app (`collection(db, 'exercises')`, `firestore().collection('exercises')` — both zero hits). iOS usage is irrelevant per project decision. **Delete all 3.** |

### Keep but relocate

| Collection | Docs | Plan |
|---|---:|---|
| `ClassAverages` | 1 | Single doc `ClassAverages/football` — actively read by 4 website components as the Premier League benchmark table. A top-level collection with a single doc is wasteful. **Move to `config/benchmarks/football`** and update 4 call sites: `components/analytics/class-comparison.tsx`, `components/analytics/class-comparison-tabbed.tsx`, `components/analytics/trends-comparison.tsx`, `components/sessions/sessions-view.tsx`. Then delete the old collection. |

### Special handling

| Collection | Docs | Plan |
|---|---:|---|
| `sessions` | 2 | Both docs are `isMergedSession: true` leftovers for the same player `XKeB9mgeNrlWHehfygiY` at the same timestamp, created 6 minutes apart on 2025-12-30 — two near-duplicate merge artifacts. The codebase has 20+ write paths to `sessions` (`createSession` in `lib/firestore.ts:1190` and 4+ read sites) but the collection is near-empty, which means the dual-write with `sensorData` isn't exercised on normal uploads. **Before killing**, verify the corresponding `sensorData/{id}` docs exist and the one non-trivial piece of data in `sessions` (a `trimConfig`) has been mirrored there. |

### Subcollections

| Path | Docs | Status |
|---|---:|---|
| `orgs/*/syncMetadata` | 2 total | Active — mobile sync marker, 1 per org |
| `users/{uid}/sessions` | 4 total | **Legacy** — only 2 users have a nested sessions subcol at all. Another abandoned breadcrumb from an earlier schema. Delete. |
| `sensorData/*/pages` | skipped | Raw 20Hz pages — the real data bloat (see below) |
| `sensorData/*/additional_data` | skipped | Duplicates top-level `precomputedAnalytics` (see below) |

---

## The `sensorData` bloat problem

`sensorData` is 1,425 docs but the real size is dominated by two arrays that get written per session:

- `derived.speed: Array<{t: number, v: number}>` — **up to ~108,000 samples** at 20 Hz for a 90-min session (~5 MB)
- `precomputedAnalytics.acceleration: Array<{t: number, a: number}>` — **another ~108,000 samples** (~5 MB)

Because a single Firestore document maxes at 1 MB, each session is physically spread across N docs in `sensorData/{id}/pages/*`. Every website read, every agent mode query, every iOS refresh fetches ALL pages.

### Redundancy to kill

1. `precomputedAnalytics.acceleration` is **trivially reconstructible** from `derived.speed` via `calculateAcceleration()` in `Cresento Website/Cresento.net/lib/utils.ts:14-73`. It costs ~10 ms client-side. Storing it triples the read size with zero benefit.
2. Pre-computed analytics are written to **TWO places** per session — `sensorData/{id}.precomputedAnalytics` (top-level field) AND `sensorData/{id}/additional_data/analytics` (subcollection doc). Pick one. The top-level field is simpler and the agent mode already reads from it.
3. Speed samples are stored at full **20 Hz**. Dropping to 10 Hz (or 5 Hz for anything but sprint detection) halves the storage and halves every read.

---

## Fatigue metrics chaos — there are actually THREE different "fatigue" fields

Extended 2026-04-11 after reading `functions/src/index.ts` and `Cresento/src/src/utils/StatsEngine.ts`:

The codebase has **three** distinct "fatigue" metrics, all called "fatigue" something, all on different scales, all produced by different code paths:

| # | Field | Scale | Where computed | Formula |
|---|---|---|---|---|
| 1 | `derived.stats.fatigueScore` | **1–10, integer** | **RN phone** in `StatsEngine.ts:computeStats()` | Maps `avgSprintLast20 / avgSprintFirst20` ratio to a 1–10 bucket via if/else if chain. `functions/src/index.ts:828-866` has the same logic for iOS. |
| 2 | `precomputedAnalytics.performance.fatigueIndex` | **0–100 float** | **Cloud Function** in `processAndSaveAnalytics()` via `calculateFatigueIndex(stats)` at `functions/src/index.ts:216-221` | `min(100, sprinting% × 1.5 + jogging% × 0.8)` — purely a zone-weighted activity score, NOT a decline metric |
| 3 | `additional_data/analytics/{id}.segmentedFatigue.score` | **0–10 float** | **Cloud Function** in `analytics-utils.ts` via `calculateSegmentedFatigue()` (subcollection write) | 8-ratio weighted average comparing peak rolling window to end window. Ratio near 1.0 → score near 10. |

**Directions of the three:**

- **#1 (sprint fatigue, 1-10):** higher = MORE fatigued. A player whose late-game sprints are 50% of early-game sprints gets score 2 ("bad"). A player who sustains sprint speed gets score 10 ("good").
- **#2 (zone fatigue index, 0-100):** higher = MORE ACTIVITY (labeled "fatigue" but is really an intensity proxy). A player sprinting 20% of the game scores 30. A benchwarmer scores 0.
- **#3 (segmented fatigue, 0-10):** higher = LESS fatigued (closer to 1.0 weighted ratio → score near 10).

**Three fields, three meanings, one of them inverted relative to the others, all called "fatigue".**

**The tool output in `lib/agent/tools.ts:869` and `:946` labels whichever one the agent picks up as `"0-10 (higher = more fatigued)"`. Sometimes that's true (#1), sometimes it's backwards (#3). The model can't tell them apart from the label alone.**

> [!danger] This is a real bug with coach-visible consequences
> In the Agent Mode demo chat that triggered the whole cleanup effort, a player (Keenan) with 0 sprints and 57% standing time scored **8.6** on one of these fatigue metrics and was described by the model as "most fatigued" — but mathematically, 8.6 on metric #3 means ~86% of peak performance retained ("barely tired"), while 8.6 on metric #1 would mean sustained sprint speed ("almost no decline"). The coach reading that chat was being told the opposite of what the data actually said.
>
> **Fix options:**
> 1. Give each metric a distinct unambiguous name (`sprintSpeedFatigueScore1to10`, `activityLoadIndex0to100`, `segmentedFatigueScore0to10`)
> 2. Unify on ONE scale direction across all three
> 3. Document the directions on each of the three storage locations AND in the Agent Mode metric-docs
>
> Until it's fixed, Agent Mode's `lib/agent/metric-docs.ts` entry for fatigue tells the model to verify scalar values via `analyze_with_code` on raw data before citing them.

Tracked for fix as part of the `sessionMetrics` migration (see [[SessionMetrics Migration Plan]]), where all three will be renamed to distinct fields on the new flat schema.

---

## Cleanup plan (sequencing)

| Phase | Risk | Est. impact | Status |
|---|---|---|---|
| **Phase 0** | None | — | **Done.** Backups captured (targeted + full metadata) |
| **Phase 1** | Low | 18 ghost docs deleted, 7 collections removed | Pending user sign-off |
| **TTL setup** | None | Auto-cleans ~200 accumulating stale docs | Pending |
| **Phase 2** | Medium | **Biggest win** — drops ~50% of sensorData read size by killing `precomputedAnalytics.acceleration` + client-side reconstruct | Pending |
| **Phase 3** | Low | Kill `sessions` collection (2 merge leftovers), remove ~500 LOC of dual-write | Pending |
| **Phase 4** | Low | Consolidate analytics storage to one location | Pending |
| **Phase 5** | Medium | Downsample speed from 20 Hz to 10 Hz on new sessions | Pending |
| Phase 6 (next sprint) | High | Move raw time-series entirely to Firebase Storage as binary | Future |

Expected outcome after Phases 1-5: session reads drop from ~5-10 MB to ~1-2 MB. Calendar, coach console, and agent queries 3-5× faster.

---

## Backup locations (today's run)

All under `Cresento Website/Cresento.net/firestore-backup/` (gitignored):

- `2026-04-11T02-36-05-353/` — initial dry run (no files written, just manifest.json)
- `2026-04-11T03-42-52-129/` — **targeted backup of 9 suspect collections** (22 docs, 0.01 MB). JSON files for every ghost-collection doc classified above.
- `2026-04-11T04-46-10-340/` — **full metadata backup** excluding `sensorData` (692 docs, 1.57 MB). Missing 11 of 12 `sessionGroupDismissals` docs due to the MAX_PATH bug that was fixed after this run. Re-run needed for a complete snapshot of that collection.

---

## Open questions for before Phase 1

1. Does the RN or iOS app read from `exercises`? If yes, the 3 exercise docs stay. If no, delete.
2. Are the 2 `sessions` collection docs' `trimConfig` mirrored in `sensorData`? If yes, safe to delete sessions entirely. If no, migrate first.
3. Which clients read `precomputedAnalytics.acceleration` directly? Website chart renderers reconstruct on demand — but if RN or iOS read it directly, they need to be updated before the field is removed.

---

## Related

- [[Firebase Backend]] — the durable inventory, paging rules, wrapper rules
- [[Data Pipeline]] — how `sensorData` gets populated
- [[Cresento Website]] — `lib/firestore.ts`, the website's Firestore wrapper
- [[Cresento React Native App]] — uses `SessionUploader.ts` for writes
- [[DataRecoveryIOS (ShinPad iOS)]] — writes most of the "mystery" collections (liveStatus, padOwners, rawdata, emailCodes)
- [[StatsEngine Cross-Platform]] — where metric formulas live; the fatigue label bug is tracked here
- [[01 - Critical Preservation Rules#📦 Firestore session paging]]
