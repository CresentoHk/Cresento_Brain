---
title: Tech Debt Audit 2026-04-14
type: audit
tags:
  - project/website
  - frontend
  - tech-debt
created: 2026-04-14
status: paused-phase-2-partial
paused: 2026-04-15
next-up: phase-2c-firestore-split-then-phase-3
---

# 🧹 Cresento Website Tech Debt Audit — 2026-04-14

Tech debt inventory for the Next.js website at `Cresento Website/Cresento.net/`. Scored on Impact (1–5), Risk (1–5), and Effort (1–5, inverted). Priority = (Impact + Risk) × (6 − Effort).

Related: [[Cresento Website]] · [[Security Audit 2026-04-14]] · [[Firestore Collection Audit 2026-04-11]] · [[01 - Critical Preservation Rules]]

> [!success] Phase 1 shipped + Phase 2 partial — 2026-04-15
> Branch `tech-debt-phase-1` off `agent-demo`. **9 atomic commits total**, `npm run build` green at every step.
> - **Phase 1**: lockfile + deps + chart consolidation + `lib/log.ts`. See [[#Phase 1 — Shipped 2026-04-15]].
> - **Phase 2a+2b+2d**: wrapper bypass cleanup (6 files), bounded `getRecentSensorDataByOrg`, `ARCHITECTURE.md`. See [[#Phase 2 — Shipped 2026-04-15]].
> - **Phase 2c (firestore.ts split)**: explicitly deferred to its own PR by user.

> [!todo] 🔖 Pick up here next session
> **Paused 2026-04-15.** Branch `tech-debt-phase-1` is committed but **not pushed** and **not merged**. Starting point for next session:
>
> 1. **Decide on the current branch**: push `tech-debt-phase-1` and open a PR against `agent-demo`, OR merge locally, OR keep iterating.
> 2. **Phase 2c — `lib/firestore.ts` split** (the big one that was deferred). See [[#Phase 2c — Deferred]] below for the plan. Suggest starting a fresh `tech-debt-phase-2c` branch off whatever base is clean at that point.
> 3. **Phase 3 — not started**. See [[#Phase 3 — Not started]] below. Biggest candidates:
>    - Extract `<AgentMode/>` from `app/coach/page.tsx` (1,609 LOC mega-component).
>    - Decompose `components/sessions/sessions-view.tsx` (6,133 LOC).
>    - Lazy-load heavy chart components.
>    - Add component/integration tests.
> 4. **Caller migration**: `app/sessions/analytics/page.tsx` still uses the deprecated `getSensorDataByOrg`. Swap it to `getRecentSensorDataByOrg({ orgId, limit: 50 })` once you confirm the page actually wants "recent" semantics.
>
> **Before editing any of the files from these phases, re-read [[01 - Critical Preservation Rules]] and the scope-boundary callout above.** The mega-components overlap with load-bearing code (agent, calendar, session state machine).

> [!warning] Scope boundary
> Do **NOT** touch anything in `lib/agent/memory/*`, `app/api/agent/route.ts` token/duration limits, session state machine logic, or anything listed in [[01 - Critical Preservation Rules]]. Phase-1 remediation is only the "safe" items below.

---

## Summary

| Category            | Items | Severity |
| ------------------- | ----- | -------- |
| Code debt           | 4     | High     |
| Architecture debt   | 5     | High     |
| Test debt           | 1     | High     |
| Dependency debt     | 4     | High     |
| Documentation debt  | 1     | Medium   |
| Performance risk    | 3     | Medium   |

**Overall severity: HIGH** — monolithic firestore layer + mega-components + ~3% test coverage.

---

## Top 10 Highest-Impact Items

| # | Item                                                                                                                                          | Category          | Priority |
| - | --------------------------------------------------------------------------------------------------------------------------------------------- | ----------------- | -------- |
| 1 | Critical deps pinned to `latest` (firebase, framer-motion, recharts, chart.js, next-themes) — any major release can break prod unannounced    | Dependency        | 45       |
| 2 | Dual lock files: `package-lock.json` + `pnpm-lock.yaml` both committed → nondeterministic CI                                                  | Dependency        | 35       |
| 3 | Layering violation: 4 components import `firebase/firestore` directly, bypassing `lib/firestore.ts` wrapper                                   | Architecture      | 32       |
| 4 | Unbounded query `getSensorDataByOrg` (`lib/firestore.ts:2077`) — `where("userID","in",playerIds)` can fan out org-wide                        | Performance/Risk  | 32       |
| 5 | Dual chart libraries shipped: `chart.js` + `react-chartjs-2` AND `recharts` (~400 KB wasted bundle)                                           | Dependency        | 24       |
| 6 | `app/coach/page.tsx` (1,609 LOC) and `components/dashboard/sessions-calendar.tsx` (1,620 LOC) — multiple responsibilities per file            | Code              | 24       |
| 7 | No architecture/API docs; README is v0.app sync placeholder; `Session` vs `SessionGroup` vs deprecated `Game`/`Drill` migration undocumented  | Documentation     | 24       |
| 8 | `components/sessions/sessions-view.tsx` mega-component (6,133 LOC, 20+ states, 30+ firestore imports)                                         | Code              | 18       |
| 9 | Test coverage ~3.3% (7 test files vs 211 source files); only `lib/analytics` is tested                                                        | Test              | 16       |
| 10| `lib/firestore.ts` god file (4,875 LOC, 100+ exports, 248 console.logs, cache + reads + writes + tracking mixed)                              | Code/Architecture | 10*      |

\* Item 10 has a low priority score only because Effort is high (5). Impact is maximal — it's load-bearing for the whole website.

---

## Detailed Findings

### 1. Code Debt

| Issue                    | File                                                      | Details                                                                                                  |
| ------------------------ | --------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| **God file**             | `lib/firestore.ts`                                        | 4,875 lines, 100+ exports. Mixes reads / writes / cache / tracking / types. 248 `console.log/error` calls. |
| **Mega component**       | `components/sessions/sessions-view.tsx`                   | 6,133 lines; 20+ local states; imports 30+ firestore functions.                                          |
| **Large components**     | `components/dashboard/sessions-calendar.tsx` (1,620), `app/coach/page.tsx` (1,609), `components/analytics/performance-overlay-chart.tsx` (1,616) | Mixed state/logic/UI; agent mode embedded in coach page.                                                 |
| **Analytics duplication**| `lib/analytics/{stats,insights,trends,baselines,goal-insights}.ts` | Overlapping aggregation / filtering; no shared util.                                                     |
| **Deprecated types live**| `lib/types.ts`                                            | `Drill` and `Game` marked `@deprecated` but still referenced in UI.                                      |

### 2. Architecture Debt

| Issue                   | Files                                                                                                                    | Impact                                                              |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------- |
| **Layering violation**  | `app/teams/page.tsx`, `components/analytics/trends-comparison.tsx`, `class-comparison.tsx`, `class-comparison-tabbed.tsx`| Import `firebase/firestore` directly. CLAUDE.md explicitly forbids. |
| **No service layer**    | —                                                                                                                        | Session grouping, trimming, analytics live in components.           |
| **Cache coupled to queries** | `lib/firestore.ts:360-450`                                                                                          | `playerCache`/`sessionCache` objects embedded in query functions; no TTL strategy. |
| **Unclear server/client split** | `app/coach/page.tsx` and peers                                                                                   | `"use client"` files importing `serverTimestamp` + 60+ firestore symbols. |
| **Analytics fragmented**| `lib/analytics/` (8 files)                                                                                              | Unclear module ownership; `data-fetcher.ts` also queries firestore directly. |

### 3. Test Debt

- **211 source files**, **7 test files** → ~3.3% coverage ratio.
- Jest threshold set to 70% in `jest.config.js` but scoped only to `lib/analytics`.
- Zero component tests, zero integration tests, zero firestore tests.

### 4. Dependency Debt

| Issue                    | Details                                                                                                   |
| ------------------------ | --------------------------------------------------------------------------------------------------------- |
| **`latest` pins**        | `firebase`, `recharts`, `chart.js`, `framer-motion`, `next-themes` all unversioned.                       |
| **Dual chart libs**      | `chart.js` + `react-chartjs-2` AND `recharts` installed. Pick one.                                        |
| **Dual lock files**      | `package-lock.json` (774 KB) + `pnpm-lock.yaml` (472 KB) both committed.                                  |
| **Radix UI scatter**     | Most `@radix-ui/*` on `latest`, a few pinned exact (e.g. `1.2.2`). Inconsistent upgrade strategy.          |

### 5. Documentation Debt

- README is an auto-generated v0.app sync placeholder.
- No JSDoc on `lib/firestore.ts` (100+ exports).
- No migration path documented for `Session` / `SessionGroup` / `Game` / `Drill`.
- Agent memory system is well-documented in `Cresento_Brain/` — this is the exception, not the rule.

### 6. Performance Risk

| Risk                  | Location                                                  | Notes                                                         |
| --------------------- | --------------------------------------------------------- | ------------------------------------------------------------- |
| **Unbounded query**   | `lib/firestore.ts:2077` (`getSensorDataByOrg`)            | `where("userID","in",playerIds)` can load org-wide data.      |
| **No memoization**    | `components/sessions/sessions-view.tsx`                   | Rebuilds 20+ handlers every render; no `useCallback`/`useMemo`. |
| **Bundle size**       | `components/analytics/performance-overlay-chart.tsx`      | 1,616 LOC of chart logic shipped in main bundle; lazy-load it.|

---

## Phased Remediation Plan

### Phase 1 — Quick wins (1 week, low risk) ✅ SHIPPED 2026-04-15

1. Delete one lock file; pin CI to `npm` **or** `pnpm`.
2. Replace `latest` pins with caret ranges (especially `firebase`, `recharts`).
3. Audit which chart lib is actually rendered; remove the other. Consolidate on one.
4. Strip `console.log` from `lib/firestore.ts`; route through a tiny `lib/log.ts` gated on `NODE_ENV`.

See **[[#Phase 1 — Shipped 2026-04-15]]** below for details.

### Phase 2 — Safe refactors (2–3 weeks, alongside features)

5. Split `lib/firestore.ts` into `lib/firestore/{reads,writes,cache,tracking,types}.ts`. Preserve exports via a barrel file so no callers change.
6. Reroute the 4 direct `firebase/firestore` imports through the wrapper.
7. Add a bounded-page variant of `getSensorDataByOrg`; migrate callers.
8. Write `ARCHITECTURE.md` covering data flow + Session/SessionGroup/Game migration.

### Phase 3 — Structural (ongoing, touch-as-you-go)

9. Decompose `sessions-view.tsx` → `<SessionsList/>`, `<SessionDetail/>`, `<SessionCalendar/>` with shared context.
10. Extract `<AgentMode/>` from `app/coach/page.tsx`.
11. Lazy-load heavy chart components.
12. Add component/integration tests whenever a mega-component is split — target 30% `components/` coverage per quarter.

---

## Do Not Touch

Covered in [[01 - Critical Preservation Rules]], reiterated here because the tempting refactor targets overlap with load-bearing code:

- `lib/agent/memory/*` — Phase 1 memory system, shipped 2026-04-11.
- `app/api/agent/route.ts` — `maxDuration = 300`, `max_tokens: 12288`, `reasoning.max_tokens: 16384` are all load-bearing.
- Session state machine (`idle → logging → flushing`).
- BLE UUIDs / characteristics / command bytes ([[BLE Protocol]]).
- Website calendar session loading — historically sensitive.

---

## Phase 1 — Shipped 2026-04-15

Branch: `tech-debt-phase-1` off `agent-demo`. 5 atomic commits. `npm run build` verified green before merge.

### Commits

| # | SHA       | Change                                                                                   |
| - | --------- | ---------------------------------------------------------------------------------------- |
| 1 | `103f526` | Removed `pnpm-lock.yaml`; standardized on npm                                            |
| 2 | `6572ffd` | Pinned 15 `latest` deps to caret ranges matching installed versions                      |
| 3 | `f0dcd46` | Migrated 3 recharts components → chart.js; removed recharts dep                          |
| 4 | `853290c` | Created `lib/log.ts`; routed 263 `console.*` calls in `lib/firestore.ts` through it     |
| 5 | `4f16e51` | Regenerated `package-lock.json` (61 transitive packages dropped)                         |

### Audit correction caught during execution

> [!info] Chart library direction was backwards in original audit
> Original audit said "drop chart.js, keep recharts." Actual usage: **chart.js in 28 files, recharts in 3 files**. Flipped direction mid-execution — kept chart.js, migrated the 3 recharts files. Document this for future audits: count usage before recommending which to drop.

### New load-bearing file

**`lib/log.ts`** — scoped logger.

- `log.debug` / `log.info` — NO-OP in production (gated on `process.env.NODE_ENV`).
- `log.warn` / `log.error` — always fire. Real signals.
- `log.scope("firestore")` — returns a logger that prefixes every line with `[firestore]`. Filterable in Vercel / browser console.
- Convention: one scope per module. Do not reintroduce bare `console.*` calls in `lib/firestore.ts` — see comment at top of that file.

### Files migrated from recharts → chart.js

Pattern used (matching existing `components/analytics/chartjs-*.tsx`):

```tsx
import { Line } from "react-chartjs-2"
import "@/lib/chart-config"                // registers chart.js components
import { ClientOnly } from "@/components/client-only"
// ...
<ClientOnly fallback={<div className="h-full w-full" />}>
  <Line data={chartData} options={chartOptions} />
</ClientOnly>
```

1. `components/dashboard/activity-chart.tsx` — Bar chart (1 chart).
2. `components/sessions/performance-charts.tsx` — Line×3, Bar (horizontal), Scatter (5 charts).
3. `components/players/player-detail-modal.tsx` — Pie → Doughnut (1 chart). Also removed ~12 dead recharts imports (RadarChart, PolarGrid, etc.) that were never rendered.

### Phase 1 impact

- **Dependencies:** No `latest` pins; single lockfile (npm); recharts removed along with 60 transitive packages.
- **Production logs:** 191 debug `console.log` calls in `lib/firestore.ts` no longer ship to prod.
- **Build:** Green, 16 static pages generated, no new warnings.
- **Untouched:** `lib/agent/memory/*`, `app/api/agent/route.ts`, session state machine, sessions-calendar, the 28 existing chart.js analytics components, all mega-components.

---

## Phase 2 — Shipped 2026-04-15

Same branch as Phase 1 (`tech-debt-phase-1`). 4 additional atomic commits. Build verified green at each step.

### Commits

| # | SHA       | Change                                                                                                            |
| - | --------- | ----------------------------------------------------------------------------------------------------------------- |
| 6 | `6cca5fd` | Added 4 wrapper functions: `getTeam`, `getUserProfile`, `getPlayerRecordsByUserUid`, `getClassAverages`           |
| 7 | `c134f38` | Routed 6 UI files through the firestore wrapper; removed their `firebase/firestore` imports                       |
| 8 | `ddeafef` | Added bounded `getRecentSensorDataByOrg`; deprecated unbounded `getSensorDataByOrg`                               |
| 9 | `e5d4ae9` | Added `ARCHITECTURE.md` covering data flow, wrapper rules, session model, agent pointers                         |

### Phase 2a — Wrapper bypass cleanup (6 files)

Fixed 6 UI-layer files that were importing `firebase/firestore` directly and bypassing `lib/firestore.ts` — explicitly forbidden by CLAUDE.md:

- `components/analytics/trends-comparison.tsx` — `ClassAverages/Football` fetch
- `components/analytics/class-comparison.tsx` — same
- `components/analytics/class-comparison-tabbed.tsx` — same
- `components/sessions/sessions-view.tsx` — 3× `ClassAverages/Football` fetches
- `app/teams/page.tsx` — team lookup + player records by uid
- `app/teams/[id]/page.tsx` — team + player records + user profile lookups

All now go through new wrapper functions. `/teams` and `/teams/[id]` bundle sizes dropped slightly with `firebase/firestore` no longer pulled in at the page level.

### Phase 2b — Bounded `getRecentSensorDataByOrg`

The original `getSensorDataByOrg` had two latent bugs:

1. **Unbounded fan-out** — loads every sensor doc for every player in the org (tens of MB on large orgs).
2. **Silent 30-player cap** — Firestore's `where("userID", "in", [...])` limit.

New function in `lib/firestore.ts`:

```ts
getRecentSensorDataByOrg({ orgId, limit?, since? })
```

- Chunks player-id array into groups of 30 and runs queries in parallel.
- Accepts per-player `limit` (default 50) and optional `since` lower bound.
- Still avoids a composite index by sorting client-side after fan-in.

Old function kept with `@deprecated` JSDoc so its one caller (`app/sessions/analytics/page.tsx`) can migrate deliberately.

### Phase 2d — `ARCHITECTURE.md`

New file: `Cresento Website/Cresento.net/ARCHITECTURE.md` (~150 lines). Covers:

- Data flow from ShinPad firmware → Firestore → wrapper → UI
- The wrapper rule (UI never imports `firebase/firestore`), with exception list
- `lib/log.ts` logging convention
- Session vs SessionGroup vs deprecated Game/Drill migration state
- Agent (Cora) entry points + load-bearing limits
- Pointers into `Cresento_Brain/` for deeper concepts

### Phase 2c — Deferred

> [!info] Full `lib/firestore.ts` split deferred
> User deliberately chose to land Phase 2a+2b+2d first and defer the 6-file split (`reads/writes/cache/tracking/types/utils`) to its own `tech-debt-phase-2c` branch. Reason: ~95 exports moving across 6 files is a much bigger blast radius than Phase 2a/2b combined, and wants isolation for review.

**Plan when you pick this up again:**

1. Start a fresh branch: `git checkout -b tech-debt-phase-2c` off `agent-demo` (or off `tech-debt-phase-1` if it's merged by then).
2. Re-run the mapping agent — the export inventory from 2026-04-15 was in a previous session's transient output and is gone. Cheap to regenerate.
3. Proposed target file layout (from the prior mapping pass):
   - `lib/firestore/reads.ts` — ~45 `get*` / `list*` / `subscribe*` functions
   - `lib/firestore/writes.ts` — ~35 `create*` / `update*` / `delete*` / `upload*` functions
   - `lib/firestore/cache.ts` — `playerCacheEnriched`, `playerCachePlain`, `sessionCache`, `sessionGroupsCache`, `missingImagesCache`, `playerFetchInFlight`, plus `clear*Cache` helpers. **Module-level state lives here, nowhere else.**
   - `lib/firestore/tracking.ts` — `perfMetrics`, `trackRead`, `trackCache`, `trackStorage`, `getPerformanceMetrics`, `resetPerformanceMetrics`, `printPerformanceReport`
   - `lib/firestore/types.ts` — re-exports of `SensorData`/`SensorDataSummary`/`SensorDataStats` from `@/lib/types`, plus the interfaces declared in this file (`OrgSyncMetadata`, `DeduplicateResult`, `PlayerSeasonalStats`, `MergeSessionsResult`, `PlayerRank`, `CoachRankings`, `WatchlistEntry`, `CoachWatchlist`, `ACWRDaySnapshot`, `PlayerACWRTimeline`, `TeamACWRTimelineResponse`)
   - `lib/firestore/utils.ts` — `getDateFromTimestamp`, `computeSessionTimes`, `convertAggregatesToSessions`, `getMonthKeysAround`, `extractSensorDataSummary`, localStorage helpers for `missingImagesCache`
   - `lib/firestore.ts` — now just a **barrel file**: `export * from "./firestore/reads"`, etc. Keeps every existing `from "@/lib/firestore"` import working.
4. **Tricky cases to watch**:
   - `ensurePlayerRecord` is a get-or-create → put in `writes.ts`.
   - `getTeamMembers` is aliased to `getPlayersByTeam` → keep the alias export.
   - `loadFullSessionDataInBackground` touches `sessionCache` → lives in `cache.ts` or imports from `cache.ts`.
   - `flog` (the scoped logger) needs to be imported into each new file separately since it's currently top-of-firestore.ts state.
   - Do not break the `@deprecated` comment on `getSensorDataByOrg`.
5. After the split, run `npm run build` and grep for any caller still importing `from "@/lib/firestore/*"` directly (they should all go through the barrel).

**Risk:** High. 20+ files import from `@/lib/firestore`. The barrel file strategy is specifically to keep their imports unchanged, but it only works if every export is re-exported correctly. A quick sanity check: `git diff --stat` on the caller files should show **zero caller changes** from a well-executed split.

### Phase 2 impact

- **Wrapper compliance**: 6 UI files no longer bypass `lib/firestore.ts`. `firebase/firestore` imports outside `lib/*` are now zero (except `lib/analytics/data-fetcher.ts` which is the analytics engine by design).
- **Reliability**: `getRecentSensorDataByOrg` fixes a silent 30-player ceiling that would throw on any org with more members.
- **Onboarding**: `ARCHITECTURE.md` gives new contributors an entry point that isn't "read 5,000 lines of firestore.ts and guess."
- **Build**: Green, bundle sizes roughly flat (`/teams` marginally smaller).

---

## Phase 3 — Not started

From the original phased plan. **Touch-as-you-go, not a sprint** — these are big and overlap with load-bearing code.

1. **Extract `<AgentMode/>`** from `app/coach/page.tsx` (1,609 LOC). Easiest Phase 3 win — agent UI is relatively self-contained; pulling it into `components/coach/agent-mode.tsx` (which already exists) and importing it should be straightforward. Check what still lives in `coach/page.tsx` that should move with it.
2. **Decompose `components/sessions/sessions-view.tsx`** (6,133 LOC) into `<SessionsList/>`, `<SessionDetail/>`, `<SessionCalendar/>` with a shared context. **Highest risk** — this file owns the calendar view, which [[01 - Critical Preservation Rules]] flags as load-bearing. Go in small slices; verify the calendar behavior after each slice.
3. **Lazy-load heavy charts** — `components/analytics/performance-overlay-chart.tsx` alone is 1,616 LOC. Wrap the analytics bundle with `next/dynamic` so it doesn't land in the `/sessions` first-load JS (currently 534 KB).
4. **Component tests** — target 30% `components/` coverage per quarter. Start with the components you split in step 1 and step 2 — they're easier to test in isolation than the mega-components they came from. Infrastructure is already there: jest + ts-jest + `__tests__/` directory.

Plus the deferred item:

5. **Full `lib/firestore.ts` split** — see [[#Phase 2c — Deferred]] above for the plan.

---

## Related

- [[Cresento Website]]
- [[Security Audit 2026-04-14]]
- [[Firestore Collection Audit 2026-04-11]]
- [[SessionMetrics Migration Plan]]
- [[01 - Critical Preservation Rules]]
