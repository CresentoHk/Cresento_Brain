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
│   ├── position-scoring.ts     (~33 KB, position-specific player benchmarks)
│   ├── trim-utils.ts           (~31 KB, session trimming)
│   └── analytics/              (advanced analytics engine)
│       ├── service
│       ├── stats
│       ├── trends
│       └── insights
├── components/
│   ├── analytics/              (35+ analytics components)
│   ├── dashboard/
│   │   └── sessions-calendar.tsx   (~70 KB, CRITICAL)
│   └── ui/                     (shadcn/ui base components)
├── contexts/
│   └── auth-context.tsx
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
