---
title: Firebase Backend
type: shared-system
tags:
  - shared
  - backend
  - firebase
  - critical
created: 2026-04-11
project: cresento-8b603
---

# 🔥 Firebase Backend

The single backend that everything talks to. **One Firebase project (`cresento-8b603`)** is shared by the [[Cresento React Native App|RN app]], [[DataRecoveryIOS (ShinPad iOS)|iOS app]], and [[Cresento Website|website]]. Changes to the data model affect ALL platforms.

> [!danger] Cross-platform impact
> Before changing a collection schema, ask: "does this break the RN decoder, the iOS decoder, AND the website's `lib/firestore.ts`?" If you don't know, the answer is yes.

---

## Services in use

- **Auth** — email/password + Google OAuth
- **Firestore** — primary data store
- **Cloud Storage** — file uploads (e.g. media)
- **Cloud Functions** — server-side logic
- **Crashlytics** — RN + iOS crash reporting (logs land in `React app crashlytics/`)

---

## Collections

> [!info] For the full current inventory see [[Firestore Collection Audit 2026-04-11]]
> That note has every collection with doc counts, code references, and cleanup status as of today. This section documents the durable reference model — what each collection is FOR.

### `users/{uid}`

- Profile (name, email, displayName)
- **`role`** — `"player"` or `"coach"` (`"admin"` exists in the auth guard but has no creation UI — set manually in Firestore). Drives navigation in all clients.
- `orgId` — the org the user belongs to
- Session counters
- Privacy settings (configured by coaches on the website, read by mobile)

### `sensorData/{docId}`

> [!warning] Paged — read paging rules below
> Each session is stored across MULTIPLE `sensorData` documents, not one. The page schema is shared and load-bearing.

- One physical recording session = parent doc + N `pages/*` subcollection docs
- Each page ≤ **700 KB** (Firestore hard limit is 1 MB; the cap reserves headroom)
- Pages are atomic batched on write so partial sessions don't appear
- Decoded by RN, iOS, AND the website — keep the schema stable
- Top-level fields: `derived.speed[]` (20 Hz raw), `derived.stats` (aggregates), `precomputedAnalytics` (Cloud-Function computed regression/fatigue/acwr/etc.), `trimConfig`
- Also has an `additional_data/analytics` subcollection that duplicates most of `precomputedAnalytics` — cleanup target, see [[Firestore Collection Audit 2026-04-11#The sensorData bloat problem]]

### `orgs/{orgId}`

- Organization (club, school) metadata
- Subcollection `syncMetadata/latest` — mobile sync marker

### `teams/{teamId}`

- Team info — formation, `coachUids[]`, orgId, optional branding image
- Note: `Team` (capital T) is a DEAD ghost collection from an older schema. See [[Firestore Collection Audit 2026-04-11|the audit]].

### `players/{playerId}`

- Player profiles — firstName, lastName, position, teamId, orgId, userUid
- Separate from `users` because a player isn't always a registered user (coaches can add players by name)
- Auto-created for user-backed players on first login via `ensurePlayerRecord` in `lib/firestore.ts:691`

### `sessionGroups/{groupId}`

- Game/training groupings — title, type, startTime, `sessionIds[]`, goals
- Used by the coach calendar + the agent mode's game analysis tools
- Auto-classifier matches raw sessions to scheduled activities and updates these

### `scheduledActivities/{activityId}`

- Planned training/game schedule for a team
- Feeds the auto-classifier that maps raw uploaded sessions to their planned slot

### `teamInvites/{inviteId}` — ⚠️ needs TTL

- Team invite links (code, teamId, role, expiresAt)
- Written by `lib/invites.ts:createTeamInvite`
- **Accumulates forever** — 128 stale docs as of 2026-04-11. Needs a TTL policy.

### `emailCodes/{code}` — ⚠️ needs TTL (iOS only)

- Email verification codes, written by iOS signup flow (`DataRecoveryIOS/.../SignUpSequence.swift`)
- Should expire after 24 h but currently doesn't — 116 stale as of 2026-04-11

### `liveStatus/{odId}` (iOS only)

- Real-time "is player X currently recording" status
- Written by iOS BLE recording, read by iOS Coach view — NOT used by the website
- Short-lived docs, tiny

### `padOwners/{stableId}` (iOS + RN only)

- Device ↔ user binding for shin-pad pairing
- Written by iOS and RN BLE contexts, read by the coach device manager in both apps
- Stable ID = BLE device stable identifier (not advertising MAC)
- NOT used by the website

### `rawdata/{sessionId}` (iOS only, metadata)

- Metadata pointer to raw CSV files uploaded to Firebase Storage at `rawdata/{userId}/{sessionId}.csv`
- The actual CSV is in Storage — this collection just holds `{ userId, sessionId, storagePath, uploadedAt, ... }`
- NOT used by the website or agent mode

### `team_monthly_sessions/{docId}`

- Pre-aggregated monthly rollups of team session data
- Written by Cloud Function `onSensorDataWrite` whenever a new session lands
- Read by the website coach console (via `coachConsoleCache` wrapper) and the agent mode's calendar pre-load path
- Critical for fast calendar loading — the fast path for "what did the team do in March"

### `coachConsoleCache/{cacheDocId}`

- Cloud-Function-computed rollups per team per period
- Powers the coach dashboard's fast load
- Should have TTL / cache invalidation logic (currently doesn't, so it can hold stale data if Cloud Function doesn't run)

### `sessionGroupDismissals/{longId}`

- Tracks which auto-classifier suggested session groups a user has dismissed from their UI
- **Doc IDs are 280+ char concatenations of multiple session IDs** — broke the initial backup script (fixed in `scripts/backup-firestore.mjs` on 2026-04-11)
- Should probably be restructured as a subcollection under `users/{uid}/dismissedGroups/{groupId}` — simpler queries, shorter IDs

### `config/{configKey}`

- System configuration — currently just `config/openrouter` (Agent Mode key pool)
- See [[Cresento Website#Agent Mode]] for the key-pool failover architecture

### `agentUsage/{uid}/daily/{date}` + `agentUsage/{uid}/events/{autoId}`

- Per-user Agent Mode usage logging, added by `lib/agent/usage-log.ts` (V2, April 2026)
- Daily doc holds atomic-increment counters (requests, promptTokens, completionTokens, toolCallCount, keyUsage map)
- Events subcollection holds immutable audit log entries — one per request
- Server-side only, written fire-and-forget from the `finally` block in `app/api/agent/route.ts`

### `ClassAverages/football` ← relocate

- Single doc holding the Premier League benchmark table (player_load, max_speed, sprint_count, etc.)
- Read as fallback data by 4 website analytics components
- **Planned to move to `config/benchmarks/football`** so it's not a top-level collection with a single doc
- See [[Firestore Collection Audit 2026-04-11]]

---

## Paging rules

> [!danger] Always go through the wrapper
> - **RN:** `Cresento/src/src/utils/SessionUploader.ts`
> - **Web:** `Cresento Website/Cresento.net/lib/firestore.ts`
> - **iOS:** Existing ViewModels / services
>
> Never write `sensorData` documents directly. Never bypass `lib/firestore.ts` from a website component.

The 700 KB cap is not a guideline, it's a structural requirement. Going over risks a runtime Firestore error in the field — and crashes happen far from the codebase that wrote them.

See [[Cresento React Native App#SessionUploader|SessionUploader]], [[Cresento Website#Library use rules]], [[01 - Critical Preservation Rules#📦 Firestore session paging]].

---

## Auth

- **Email/password** sign-in
- **Google OAuth** sign-in
- Role determines which navigation shell loads on each client
- Privacy settings are coach-controlled and cascade down to player visibility

> [!warning] Account creation
> Per global safety rules: **never create accounts on a user's behalf**. Always direct the user to do it themselves.

---

## Wrapper-only access (rule)

Each client has ONE blessed wrapper for Firestore. Use it.

| Platform | Wrapper                                                       |
| -------- | ------------------------------------------------------------- |
| Web      | `Cresento Website/Cresento.net/lib/firestore.ts` (~4600 lines)|
| RN       | `Cresento/src/src/utils/SessionUploader.ts` + utils           |
| iOS      | Existing ViewModels (`BLEViewModel`, `AuthViewModel`, etc.)   |

If your new feature needs a query that doesn't exist in the wrapper, **add it to the wrapper**. Don't bypass.

---

## Related

- [[Cresento React Native App]]
- [[Cresento Website]]
- [[DataRecoveryIOS (ShinPad iOS)]]
- [[Data Pipeline]] — how `sensorData` gets populated
- [[01 - Critical Preservation Rules#📦 Firestore session paging]]
