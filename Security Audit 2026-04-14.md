---
title: Security Audit 2026-04-14
type: audit
tags:
  - project/website
  - backend
  - critical
  - security
created: 2026-04-14
status: open
---

# 🔐 Cresento Website Security Audit — 2026-04-14

Security audit of the Next.js website at `Cresento Website/Cresento.net/`, deployed on Vercel and backed by Firebase project `cresento-8b603`. Findings are prioritized by severity and exploitability.

> [!danger] Immediate action required
> Three live credentials are committed to `.env.local` — OpenRouter key, Together AI key, and a Firebase Admin RSA private key. **Revoke and rotate before doing anything else.** Assume they are compromised.

Related: [[Cresento Website]] · [[Firebase Backend]] · [[Agent Memory System]] · [[01 - Critical Preservation Rules]]

---

## Summary

| Severity | Count |
| -------- | ----- |
| Critical | 3     |
| High     | 3     |
| Medium   | 5     |
| Low      | 2     |

**Fix order for today:**
1. Revoke + rotate the three leaked keys in `.env.local`.
2. Add auth guards to the three `players/[id]/*` API routes.
3. Tighten `storage.rules` (team images + catch-all).
4. Re-enable TS/ESLint in `next.config.mjs`.
5. Lock down org creation + add field validation in `firestore.rules`.
6. Add rate limiting to public routes.

---

## 🔴 CRITICAL

### 1. Secrets committed to `.env.local`

- **File:** `Cresento Website/Cresento.net/.env.local`
- **Severity:** Critical

The file contains full, live credentials:
- `OPENROUTER_API_KEY` (`sk-or-v1-…`)
- `TOGETHER_API_KEY` (`tgp_v1_…`)
- `FIREBASE_ADMIN_PRIVATE_KEY` (complete RSA private key, `BEGIN/END` markers)

> [!danger] Impact
> Anyone with repo access can impersonate Firebase Admin — read/write/delete **all** Firestore data, bypass all security rules, exfiltrate every user's sessions, and drain LLM billing on both OpenRouter and Together.

**Fix:**
1. Revoke all three keys in OpenRouter, Together AI, and Firebase consoles **now**.
2. Rotate to new keys, set them only in Vercel env vars.
3. Purge from git history (`git filter-repo` or BFG).
4. Verify `.env.local` and `.env*` are in `.gitignore`.

---

### 2. Unauthenticated player analytics endpoints

- **Files:**
  - `app/api/players/[id]/compare/route.ts:16-96`
  - `app/api/players/[id]/insights/route.ts:17-91`
  - `app/api/players/[id]/progress/route.ts:16-84`
- **Severity:** Critical

All three accept `playerId` and `orgId` as query params, call `getAnalyticsDataForPlayerFast()`, and return performance data (speed, acceleration, fatigue, trends, AI insights) with **no auth check**. Only presence of `orgId` is validated.

**Attack:** `GET /api/players/<anyId>/progress?orgId=<anyOrg>` returns 28 days of athletes' biometric/performance data to anyone on the internet.

**Fix:** Verify the Firebase ID token (same pattern as `app/api/agent/route.ts:321-374`) and confirm the decoded user's `orgId` matches the query param before calling the fetcher.

---

### 3. Public read on all team images

- **File:** `storage.rules:26-29`
- **Severity:** Critical

```
match /teamImages/{allPaths=**} { allow read: if true; }
```

Anyone on the internet can enumerate and download every team image.

**Fix:**
```
allow read: if isAuthenticated() && getUserOrgId() == resource.metadata.orgId;
```
Or serve via signed URLs / CDN if these must be reachable unauthenticated.

---

## 🟠 HIGH

### 4. Storage catch-all rule is far too permissive

- **File:** `storage.rules:61-63`
- **Severity:** High

```
match /{allPaths=**} { allow read, write: if isAuthenticated(); }
```

Any authenticated Firebase user (including someone from a different org, or a free-tier account spun up against the project) can upload arbitrary files to any uncovered path — quota exhaustion, malicious file hosting, covert data dumps.

**Fix:** Remove the catch-all `write`. Explicitly enumerate writable paths with org/role scoping.

---

### 5. Build-time type/lint checks disabled

- **File:** `next.config.mjs:3-8`
- **Severity:** High

```js
eslint: { ignoreDuringBuilds: true },
typescript: { ignoreBuildErrors: true },
```

Hides real bugs (unhandled promises, unvalidated inputs, wrong auth comparisons) that TypeScript and ESLint would catch at build time.

**Fix:** Set both to `false`, then fix the real errors.

---

### 6. Org creation allows role escalation

- **File:** `firestore.rules:42-46`
- **Severity:** High

```
allow create: if isAuthenticated();
```

Any user can create an `orgs/*` doc. No owner field, no admin check, no field validation on updates — a malicious coach can self-promote.

**Fix:** Restrict create to admins only. Add field-diff validation so `pricingTier`, `admins`, etc. cannot be mutated via writes.

---

## 🟡 MEDIUM

### 7. No rate limiting on public API routes

`app/api/agent/route.ts` rate-limits per user, but the `players/[id]/*` endpoints (and others) don't. Combined with #2, a trivial script drains Firestore read quota and runs up a large bill.

**Fix:** Add IP + user-keyed rate limiting (Upstash/Redis or an in-memory LRU per instance).

---

### 8. Firestore write rules don't validate field types/shape

- **File:** `firestore.rules` (players/sessions/metrics write blocks around lines 78–96)

Rules only check role + org membership, not that `name is string`, `teamId is string`, metric fields are numeric, etc. A malicious coach can write malformed docs and crash downstream analytics.

**Fix:** Add `request.resource.data.X is string` assertions to each write rule.

---

### 9. Analytics data-fetcher trusts caller

- **File:** `lib/analytics/data-fetcher.ts:382-422`

`getAnalyticsDataForPlayerFast(playerId, orgId)` doesn't re-verify `player.orgId === orgId`. Defense-in-depth is missing — if any caller forgets to scope, data leaks across orgs.

**Fix:** Assert `player.orgId === orgId` inside the fetcher itself.

---

### 10. Agent code sandbox uses Node `vm`, not `isolated-vm`

- **File:** `lib/agent/code-sandbox.ts:19,153-156`

Node's `vm` is not a real security boundary. Currently only LLM-generated code runs there, so the blast radius is limited — but if user snippets are ever accepted, you must switch to `isolated-vm`. The source comment already notes this.

**Fix:** Leave for now. Add a lint rule or CI gate that fails if user-controlled strings ever reach `runSandbox`. See [[Agent Memory System]] for the agent-tool data-flow rule (tool results never flow through chat context).

---

### 11. Firestore rule helpers rely on `get()` without null-safety

- **File:** `firestore.rules:9-39` (`getUserRole`, `getUserOrgId`, `isTeamCoach`, etc.)

If a referenced doc is missing/deleted, behavior is unpredictable. Repeated rule-eval `get()` calls against nonexistent docs also burn read quota.

**Fix:** Guard with `exists(...)` and explicit defaults. Consider caching role in custom claims to avoid the extra read per rule evaluation.

---

## 🟢 LOW

### 12. Error handling leakage potential

- **Files:** `app/api/players/[id]/{compare,insights,progress}/route.ts:89-93`

Catch blocks log via `console.error` with full context — fine for Vercel logs, but if forwarded to Sentry/etc., scrub `orgId`/`userId` first.

### 13. Verify `NEXT_PUBLIC_FIREBASE_*` contains only public config

`projectId`, `authDomain`, web `apiKey` — all meant to be public. Double-check nothing sensitive is prefixed `NEXT_PUBLIC_`.

---

## Related

- [[Cresento Website]]
- [[Firebase Backend]]
- [[Firestore Collection Audit 2026-04-11]]
- [[Agent Memory System]]
- [[01 - Critical Preservation Rules]]
