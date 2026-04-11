---
title: Cresento Website
type: project
tags:
  - project/website
  - frontend
  - web
created: 2026-04-11
status: active
location: "Cresento Website/Cresento.net/"
---

# 🌐 Cresento Website (Coach Dashboard)

The Next.js coach dashboard. Coaches log in, see all their players, drill into individual sessions, configure privacy, and review analytics + AI insights.

> [!warning] Calendar view is sacred
> The sessions calendar (`components/dashboard/sessions-calendar.tsx`, ~70 KB) is a critical user-facing path and one of the largest single files in the codebase. Read [[01 - Critical Preservation Rules#🗓️ Calendar view session loader|the rule]] before touching it.

---

## Stack

- **Next.js** 14.2.16 (App Router)
- **TypeScript**
- **Tailwind CSS v4**
- **Radix UI** + **shadcn/ui**
- **Framer Motion** for animation
- **Chart.js** + **Recharts** for analytics charts
- **Firebase** (same project as RN/iOS)
- **Vercel** for deployment

---

## Layout

```
Cresento Website/Cresento.net/
├── lib/
│   ├── firestore.ts            (~4600 lines — THE Firestore wrapper)
│   ├── types.ts                (shared TS types)
│   ├── auth.ts                 (auth functions)
│   ├── firebase-admin.ts       (lazy Admin SDK singleton for server code)
│   ├── position-scoring.ts     (~33 KB, position-specific player benchmarks)
│   ├── trim-utils.ts           (~31 KB, session trimming)
│   ├── analytics/              (advanced analytics engine)
│   │   ├── service
│   │   ├── stats
│   │   ├── trends
│   │   └── insights
│   └── agent/                  (Agent Mode V3 — see "Agent Mode" below)
│       ├── config.ts           (model + system prompt + reasoning budget)
│       ├── tools.ts            (17 tool defs + executors)
│       ├── metric-docs.ts      (rich sports-science metric documentation)
│       ├── chart-renderer.ts   (server-side Chart.js via QuickChart)
│       ├── quickchart.ts       (HTTP client that replaced chartjs-node-canvas)
│       ├── code-sandbox.ts     (Node vm sandbox for analyze_with_code)
│       ├── auth-guard.ts       (Firebase ID token + role check)
│       ├── rate-limit.ts       (per-uid in-memory limiter)
│       ├── key-pool.ts         (multi-key OpenRouter failover)
│       └── usage-log.ts        (fire-and-forget usage logging)
├── app/
│   └── api/
│       └── agent/route.ts      (SSE endpoint for Agent Mode)
├── components/
│   ├── analytics/              (35+ analytics components)
│   ├── coach/
│   │   └── agent-mode.tsx      (Agent Mode chat UI)
│   ├── dashboard/
│   │   └── sessions-calendar.tsx   (~70 KB, CRITICAL)
│   └── ui/                     (shadcn/ui base components)
├── contexts/
│   └── auth-context.tsx
├── scripts/
│   ├── seed-openrouter-config.mjs    (seeds the Agent Mode key pool)
│   ├── verify-openrouter-config.mjs  (pings each key, prints status)
│   ├── backup-firestore.mjs          (full DB backup to local JSON)
│   ├── restore-firestore.mjs         (inverse, requires --yes)
│   ├── delete-ghost-collections.mjs  (hardcoded allowlist of deletable ghost cols)
│   ├── verify-and-migrate-sessions.mjs (legacy sessions orphan checker)
│   └── ttl-cleanup.mjs               (batch-delete past-expiration docs, teamInvites + emailCodes)
└── app/                        (Next.js App Router pages)
```

---

## Build commands

```bash
cd "Cresento Website/Cresento.net"
npm run dev      # local dev server
npm run build    # production build
```

Deployed via Vercel.

---

## Key files (cheat sheet)

| File                                            | Purpose                                                 |
| ----------------------------------------------- | ------------------------------------------------------- |
| `lib/firestore.ts`                              | All DB operations. Use it. Never write raw queries.     |
| `lib/types.ts`                                  | Shared types — keep aligned with mobile / iOS           |
| `lib/auth.ts`                                   | Sign-in / sign-out / role checks                        |
| `lib/position-scoring.ts`                       | Premier-League-derived position benchmarks              |
| `lib/trim-utils.ts`                             | Session trim (clip start/end of recording)              |
| `lib/analytics/`                                | Stats, trends, insights services                        |
| `components/dashboard/sessions-calendar.tsx`    | The session calendar view (70 KB, very sensitive)       |
| `components/analytics/`                         | All chart + insight components                          |
| `components/ui/`                                | shadcn primitives — extend, do not rewrite              |
| `contexts/auth-context.tsx`                     | Auth state for client components                        |

---

## Sessions calendar

`components/dashboard/sessions-calendar.tsx` is the **primary entry point** for a coach reviewing player workouts. It:

- Reads paged sessions via `lib/firestore.ts`
- Groups them by date / player / team
- Pairs with `lib/trim-utils.ts` for clip operations
- Drives drill-down into individual session analytics

Because of its size and centrality, **never refactor "while you're in there"**. Make the smallest possible change.

---

## Agent Mode (V3 — April 2026)

Agent Mode is the coach-facing LLM that answers sports-analytics questions by autonomously calling tools over the Cresento data model. The architecture deliberately separates five concerns into dedicated modules so each can be swapped independently.

### Pipeline

1. **Client** (`components/coach/agent-mode.tsx`) — chat UI, SSE consumer, renders a `PlanCard` + `DebugPanel` + streamed markdown answer. Attaches a Firebase ID token via `Authorization: Bearer <token>` on every request.
2. **Route handler** (`app/api/agent/route.ts`) — the SSE endpoint. Runs auth → rate limit → orgId check → calendar pre-load → tool loop → usage logging in that order. Emits SSE events (`thinking`, `token`, `tool_call`, `tool_result`, `chart_image`, `plan_proposed`, `ask_user`, `done`, `error`).
3. **OpenRouter** with `qwen/qwen3.6-plus` — 1M context, native vision, tool calling, reasoning. Uses `reasoning: { max_tokens: 8192 }` for deep thinking.
4. **Tools** (`lib/agent/tools.ts`) — 17 tools split into planning (plan_analysis, describe_metrics, search_metric_docs, ask_coach), discovery (get_team_roster, get_teams, list_player_sessions, get_recent_sessions, browse_calendar, query_team_overview), targeted data (load_game_data, query_game_metrics, query_session_metrics, query_metric_summary, query_player_insights, compare_players), deep analysis (get_session_timeseries, analyze_with_code), and charts (browse_charts, render_chart).
5. **Metric docs** (`lib/agent/metric-docs.ts`) — rich documentation for every sports metric (formula, inputs, interpretation, normal range, gotchas). The model reads this before citing numbers so it doesn't hallucinate.
6. **Code sandbox** (`lib/agent/code-sandbox.ts`) — Node `vm` wrapper with strict global allowlist, 5-second timeout, captured stdout. Lets the model run JavaScript against the FULL 20Hz raw time-series without paying token cost on the data.
7. **Chart rendering** (`lib/agent/chart-renderer.ts` + `lib/agent/quickchart.ts`) — builds Chart.js configs, POSTs to QuickChart.io, returns base64 PNG. Previously used `chartjs-node-canvas` which broke on Vercel because its native `canvas` dependency isn't available at runtime.

### Security layers

All enforced in `app/api/agent/route.ts` before OpenRouter is called:

1. **`verifyAgentRequest`** (`lib/agent/auth-guard.ts`) — verifies Firebase ID token via Admin SDK, loads `users/{uid}`, enforces role ∈ {coach, admin}. 401/403 on failure.
2. **`checkAndIncrement`** (`lib/agent/rate-limit.ts`) — per-uid in-memory limiter (30/hour, 200/day). 429 with `X-RateLimit-Remaining-*` headers.
3. **orgId cross-check** — rejects if `body.orgId ≠ auth.orgId` (prevents one coach from probing another team).
4. **`executeWithKeyFailover`** (`lib/agent/key-pool.ts`) — loads keys from `config/openrouter` Firestore doc (60s cache), tries in priority order, fails over on 401/403/429/5xx BEFORE any token reaches the client.
5. **`logAgentUsage`** (`lib/agent/usage-log.ts`) — fire-and-forget writes to `agentUsage/{uid}/daily/{date}` + events subcollection.

### The "don't hallucinate" rules

The system prompt enforces five hard rules that came directly from a production failure where the model claimed "you should have subbed Keenan at minute 130" based purely on dividing his final fatigue score by his total minutes played:

1. **Plan first** — `plan_analysis` is mandatory for any query containing "when", "why", "trend", "compare", "should I", "better", "worst", or a time window
2. **Understand the metrics** — call `describe_metrics` for any metric about to be cited
3. **Compute, don't estimate** — every numerical claim must come from a tool
4. **Time-based questions need time-series data** — use `get_session_timeseries` or `analyze_with_code` on raw data
5. **Sanity-check surprising results** — when a scalar contradicts expectation (especially fatigue scores), verify via code

See [[01 - Critical Preservation Rules#🧮 StatsEngine consistency across platforms|the segmentedFatigueScore label bug]] for why rule 5 exists.

### Deployment requirements

- Vercel env vars: `FIREBASE_ADMIN_PROJECT_ID`, `FIREBASE_ADMIN_CLIENT_EMAIL`, `FIREBASE_ADMIN_PRIVATE_KEY`
- Firestore `config/openrouter` doc with at least one OpenRouter key (seeded via `scripts/seed-openrouter-config.mjs`)
- `firebase-admin@12.7.0` in dependencies (was added April 2026)
- NO more `canvas` or `chartjs-node-canvas` — removed because `canvas@3.x` needs native Cairo/Pango that doesn't exist on Vercel's Amazon Linux 2023 runtime

### Hands off

- `lib/agent/metric-docs.ts` — single source of truth for what each sports metric means. Edits must stay accurate because the model cites this back to coaches.
- `lib/agent/code-sandbox.ts` — the allowlist of exposed globals is deliberately minimal. Adding `fetch`, `require`, `process`, `fs`, `setTimeout`, `globalThis`, or `eval` would break the isolation.
- The system prompt in `lib/agent/config.ts` — rules 1-5 are the result of empirical failures. Don't soften them.

---

## Library use rules

- **Always go through `lib/firestore.ts`** — never call Firestore directly from a component or page
- **Extend, don't bypass** — if you need a new query, add it to the wrapper
- **Match existing patterns** — App Router server vs client component split, the wrapper's naming, the analytics service shape

---

## 🛑 Hands off

- `lib/firestore.ts` — the wrapper everyone depends on. Changes here cascade everywhere.
- `components/dashboard/sessions-calendar.tsx` — see above
- `lib/trim-utils.ts` — pairs with calendar; trims are persisted and other clients read them
- `lib/position-scoring.ts` — benchmarks are tuned, not arbitrary

---

## Related

- [[Cresento React Native App]] — mobile client to the same backend
- [[DataRecoveryIOS (ShinPad iOS)]]
- [[Firebase Backend]] — the data model this consumes
- [[StatsEngine Cross-Platform]] — `lib/analytics/` is one of three impls
