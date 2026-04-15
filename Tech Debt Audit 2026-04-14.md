---
title: Tech Debt Audit 2026-04-14
type: audit
tags:
  - project/website
  - frontend
  - tech-debt
created: 2026-04-14
status: open
---

# 🧹 Cresento Website Tech Debt Audit — 2026-04-14

Tech debt inventory for the Next.js website at `Cresento Website/Cresento.net/`. Scored on Impact (1–5), Risk (1–5), and Effort (1–5, inverted). Priority = (Impact + Risk) × (6 − Effort).

Related: [[Cresento Website]] · [[Security Audit 2026-04-14]] · [[Firestore Collection Audit 2026-04-11]] · [[01 - Critical Preservation Rules]]

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

### Phase 1 — Quick wins (1 week, low risk)

1. Delete one lock file; pin CI to `npm` **or** `pnpm`.
2. Replace `latest` pins with caret ranges (especially `firebase`, `recharts`).
3. Audit which chart lib is actually rendered; remove the other. Consolidate on one.
4. Strip `console.log` from `lib/firestore.ts`; route through a tiny `lib/log.ts` gated on `NODE_ENV`.

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

## Related

- [[Cresento Website]]
- [[Security Audit 2026-04-14]]
- [[Firestore Collection Audit 2026-04-11]]
- [[SessionMetrics Migration Plan]]
- [[01 - Critical Preservation Rules]]
