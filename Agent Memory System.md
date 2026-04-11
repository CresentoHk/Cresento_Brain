---
title: Agent Memory System
type: shared-system
tags:
  - project/website
  - backend
  - agent
  - ml
created: 2026-04-11
status: phase-1-live
location: "Cresento Website/Cresento.net/lib/agent/memory/"
aliases:
  - Cora Memory
  - Agent Memory
---

> [!success] Phase 1 is live and verified
> Smoke-tested end to end on 2026-04-11 — episodes are being written, sessions checkpointed, conversations persisted, and recall via vector search returns the expected matches. Top-similarity of 0.9720 against the exact write query (correct behaviour for e5 query/passage prefix split). See `scripts/memory-smoke-test.mjs` to re-run.

# 🧠 Agent Memory System (Cora)

Long-term memory for the website's Agent Mode (`Cora`). Persists conversation turns as **episodes** in Firestore, semantically recalls similar past questions on each new request, and injects the most relevant ones into the system prompt so Cora doesn't re-derive analysis she's already done.

> [!info] Status: Phase 1 scaffolded, route integration pending
> All files under `lib/agent/memory/` exist. The `app/api/agent/route.ts` edits that wire memory into the request flow are documented in `lib/agent/memory/README.md` but **not yet applied** — needs manual review + apply.

---

## Why this exists

Three separate problems, same solution directory:

### Problem 1 — "Fresh instance, no recollection" bug
The original agent is fully stateless. Every POST `/api/agent` is a cold serverless invocation. When a request times out mid-tool-loop (common on Vercel's 60s default), all accumulated state (tool calls, tool results, partial assistant text) vanishes. The client asks to "continue" and the model honestly answers "I have no prior recollection" — because the server doesn't.

**Fixed by:** `sessions.ts` — checkpoints the tool-loop messages array to `agentSessions/{sessionId}` after every iteration. Next request with same sessionId resumes from the checkpoint.

### Problem 2 — No semantic recall of past Q&A
Coaches ask the same questions repeatedly ("when to sub X", "who's at risk"). Prior analyses get re-derived from scratch every time. Wastes tokens and time.

**Fixed by:** `store.ts` + `embed.ts` — records every successful turn as an `Episode` with a Together-generated embedding, recalled by cosine similarity on the next similar question, and injected into the system prompt.

### Problem 3 — Playbook discovery
By logging every turn + tool sequence + final answer, we can later data-mine which tool recipes actually work for which intents, and promote them to reusable playbooks (Phase 3).

**Fixed by:** the episode log itself is the raw data for Phase 3.

---

## Architecture

```
┌─────────────────────────────────────────────────────┐
│ app/api/agent/route.ts                              │
│                                                     │
│   export const maxDuration = 300  ← raise Vercel   │
│                                                     │
│   POST /api/agent                                   │
│     ├─ verify auth (+ displayName for attribution)  │
│     ├─ rate limit                                   │
│     ├─ parse body { messages, orgId, sessionId }    │
│     │                                               │
│     ├─ ▶ loadSession(sessionId, uid, orgId)         │
│     │     └─ resume if in-flight                    │
│     │                                               │
│     ├─ ▶ memory.recall(orgId, lastUserMsg)          │
│     │     └─ only on fresh turns, not resumes       │
│     │     └─ org-wide, attributed by coachName      │
│     │                                               │
│     ├─ tool-use loop                                │
│     │    └─ ▶ checkpointSession() after each iter   │
│     │                                               │
│     └─ finally:                                     │
│         ├─ logAgentUsage                            │
│         ├─ if success:                              │
│         │    ├─ markSessionComplete                 │
│         │    ├─ memory.recordEpisode                │
│         │    └─ upsertConversation                  │
│         └─ if fail: checkpointSession(isComplete=F) │
└─────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────┐
│ lib/agent/memory/                                   │
│                                                     │
│  config.ts        model + thresholds                │
│  interface.ts     types (Episode, Fact, ...)        │
│  embed.ts         Together AI embeddings            │
│  store.ts         FirestoreAgentMemory              │
│  conversations.ts UI-facing transcript              │
│  sessions.ts      mid-turn tool-loop checkpoint     │
│  index.ts         getMemory() singleton             │
│  README.md        full integration diff             │
└─────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────┐
│ Firestore (firebase-admin, server-side only)        │
│                                                     │
│  agentMemory/{orgId}/episodes/{autoId}              │
│  agentMemory/{orgId}/facts/{autoId}       (Phase 2) │
│  agentMemory/{orgId}/playbooks/{autoId}   (Phase 3) │
│  agentConversations/{sessionId}  UI transcript      │
│  agentSessions/{sessionId}       resumption state   │
└─────────────────────────────────────────────────────┘
```

Mirrors the existing `agentUsage/{uid}/...` pattern — same nesting depth, same admin-SDK path. See [[Firebase Backend]].

---

## Provider split: OpenRouter vs Together AI

> [!warning] Two LLM providers, used for different things
> - **OpenRouter** — all LLM chat/tool-use traffic (unchanged). `OPENROUTER_API_KEY`.
> - **Together AI** — **embeddings only**, via `intfloat/multilingual-e5-large-instruct`. `TOGETHER_API_KEY`.
>
> If you ever add new LLM features, route them through OpenRouter. Together's role is strictly vector generation.

Together was picked because:
- Embedding pricing is fractions of a cent per 1k tokens
- The multilingual e5 model handles coaches whose first language isn't English
- We already had the key

### Embedding model details

- **Model:** `intfloat/multilingual-e5-large-instruct`
- **Dimensions:** 1024
- **Distance:** COSINE
- **Prefix contract:** e5 models need `query: ` for search queries and `passage: ` for indexed text — the embed module handles this automatically via `embedQuery()` / `embedPassage()`

---

## Firestore collections

### `agentMemory/{orgId}/episodes/{autoId}`

One per Q&A turn. The thing that gets recalled.

```ts
{
  uid, orgId, sessionId,
  userQuery,       // embedded as "passage: ..."
  finalAnswer,
  toolCallSummary: [{ name, argsDigest }],
  model, promptTokens, completionTokens, durationMs,
  success, errorMessage?,
  createdAt,
  embedding: vector(1024)
}
```

### `agentConversations/{sessionId}`

Literal transcript for cross-session resume. Separate from episodes because this is what a "reopen old conversation" UI reads — no embedding, no retrieval, just the messages. Trimmed to `CONVERSATION_MAX_MESSAGES` and clipped per message.

```ts
{
  uid, orgId, sessionId, title,
  messages: [{ role, content, timestamp }],
  turnCount, createdAt, updatedAt
}
```

Keyed by `sessionId` (client-generated) so Firestore security rules can allow a user to read their own conversations by uid match, without needing the orgId in the path.

### `agentSessions/{sessionId}`

> [!warning] This is what fixes the "fresh instance" bug
> Raw OpenRouter-format tool-loop state, persisted after every tool iteration. Used for mid-turn resumption when a request times out or a client disconnects.

```ts
{
  uid, orgId, sessionId,
  messages: [ { role, content, tool_calls?, tool_call_id?, name? } ],
  iteration,          // how many tool rounds completed
  isComplete,         // true once Cora emits a final (non-tool) response
  totalPromptTokens, totalCompletionTokens,
  lastError?,
  createdAt, updatedAt
}
```

**Why separate from `agentConversations`?**
- `agentConversations` is trimmed, titled, and designed for a UI subscribe (small docs, fast reads for the "recent conversations" sidebar)
- `agentSessions` holds the full tool-call + tool-result interleave in OpenAI chat format — needed to resume the loop, useless to the UI

**When a new request arrives with an existing `sessionId`:**
- If the session is `isComplete: true` → treat as a new turn on top of the completed conversation
- If the session is `isComplete: false` → load the checkpoint, overwrite the system message with today's fresh one (date + calendar context may have changed), append any new user messages from the client that aren't already in the stored tail, resume the tool loop from where it died

Access control: the session doc stores `uid` + `orgId`, and `loadSession()` rejects any request whose auth doesn't match — defense in depth beyond Firestore rules.

### `agentMemory/{orgId}/facts/{autoId}` — Phase 2 (not built)

Extracted durable facts: "this org tracks sprint distance and ACWR", "they've had 3 injuries in the last 2 months", etc.

### `agentMemory/{orgId}/playbooks/{autoId}` — Phase 3 (not built)

Cached tool sequences keyed by intent. Injected as hints when a new query matches an existing intent with similarity > 0.85.

---

## Required Firestore indexes

One-time setup. Firebase will print a console URL on first `findNearest()` call — click it.

**Vector index:**
- Collection group: `episodes`
- Field: `embedding`
- Dimension: `1024`
- Distance: `COSINE`

**Composite filter index:**
- Collection group: `episodes`
- Fields: `success` (ASC), `createdAt` (ASC)

See [[Firebase Backend]].

---

## Required environment variables

```
TOGETHER_API_KEY=<Together AI key>
```

Already present: `OPENROUTER_API_KEY`, `FIREBASE_ADMIN_*`.

Set these locally (`.env.local`) **and** in Vercel project settings for production.

---

## Scoping: per-org with per-user attribution

> [!info] Every coach in the same org sees every other coach's past questions.
> This is deliberate — the point is that the org's collective memory compounds. Cora must always show *which coach* asked, so coaches can follow up or give credit.

- **Recall scope:** org-wide. No `uid` filter by default.
- **Attribution:** every episode stores `uid` + `coachName` (cached from Firebase Auth displayName at write time, so recall doesn't need a secondary lookup).
- **Session resumption:** strictly per-user. A coach can only resume their own in-flight sessions — `loadSession()` enforces uid match.
- **Conversation transcripts:** strictly per-user. Coach A can't read Coach B's transcripts via `agentConversations`.

The shared thing is the *long-term memory layer*. In-flight state and literal transcripts stay private.

## Integration status

| Piece                                      | Status                                     |
| ------------------------------------------ | ------------------------------------------ |
| `lib/agent/memory/config.ts`               | ✅ Live                                    |
| `lib/agent/memory/interface.ts`            | ✅ Live (Episode has `coachName`)          |
| `lib/agent/memory/embed.ts`                | ✅ Live (Together AI, e5 prefix split)     |
| `lib/agent/memory/store.ts`                | ✅ Live (pure vector search, JS filters)   |
| `lib/agent/memory/conversations.ts`        | ✅ Live                                    |
| `lib/agent/memory/sessions.ts`             | ✅ Live                                    |
| `lib/agent/memory/index.ts`                | ✅ Live                                    |
| `lib/agent/memory/README.md`               | ✅ Live (integration diff)                 |
| `lib/agent/auth-guard.ts` returns displayName | ✅ Applied                               |
| `lib/agent/usage-log.ts` undefined-field fix | ✅ Applied (pre-existing bug, unrelated) |
| `app/api/agent/route.ts` (9 edits + maxDuration 300) | ✅ Applied                       |
| `components/coach/agent-mode.tsx` sessionId | ✅ Applied                                |
| `TOGETHER_API_KEY` in `.env.local`         | ✅ Set                                     |
| `TOGETHER_API_KEY` on Vercel               | ✅ Set (confirmed by user)                 |
| `FIREBASE_ADMIN_*` in `.env.local`         | ✅ Set (this is what fixed the 401)        |
| Firestore vector index on `episodes.embedding` | ✅ Deployed via `firebase deploy`      |
| `scripts/memory-smoke-test.mjs`            | ✅ Written + smoke-tested, PASS            |
| Fact extraction (Phase 2)                  | ❌ Not built                               |
| Playbooks (Phase 3)                        | ❌ Not built                               |
| Plan→confirm→execute workflow              | ❌ Not built (see below)                   |

The full integration diff is in `lib/agent/memory/README.md`.

---

## Plan → Confirm → Execute workflow (future)

The user asked for a workflow where Cora proposes a plan, the user approves/edits, and only then does she execute. **This is not built yet** because it requires:

1. Server: after `plan_analysis` tool call, close the SSE stream with a new event type (`plan_awaiting_confirmation`) instead of auto-continuing — similar to how `ask_coach` already works
2. Client: render the plan as an editable checklist, send back a `planApproved` or `planRevised` field in the next POST
3. Server: if `planApproved` is in the request body, skip the `plan_analysis` step on the next turn and proceed directly to execution

The existing `plan_proposed` SSE event and `plan_analysis` tool are the right primitives — the change is behavioral, not structural. Design it after Phase 1 ships so you can see which plans coaches accept vs revise.

---

## Gotchas discovered during Phase 1 bring-up

These are the traps that cost real time. Capturing so they don't bite twice.

### 1. `FIREBASE_ADMIN_*` env vars were missing locally

> [!warning] Misleading error: "Invalid or expired ID token"
> The symptom was a 401 from `/api/agent` claiming the Firebase ID token was bad. It wasn't. The real cause was that `lib/firebase-admin.ts` couldn't initialize because `FIREBASE_ADMIN_PROJECT_ID`, `FIREBASE_ADMIN_CLIENT_EMAIL`, and `FIREBASE_ADMIN_PRIVATE_KEY` were missing from `.env.local`. The thrown init error gets caught by `auth-guard.ts:48` and re-thrown as the generic "invalid token" message.

Fix: always make sure all three `FIREBASE_ADMIN_*` vars are in `.env.local` locally, not just in Vercel. Production was working because Vercel had them set; local didn't.

### 2. Firestore rejects `undefined` field values

The existing `lib/agent/usage-log.ts` was silently failing every successful request because it spreads the event object (which has `error?: string`) into a Firestore doc. On a successful run `error` is `undefined`, and Firestore throws `Cannot use "undefined" as a Firestore value`.

> [!info] Rule for all Firestore writes from this project
> Never spread an object with optional fields directly into a Firestore payload. Either:
> 1. Build the payload field-by-field, skipping undefined values, OR
> 2. Filter with `Object.entries(x).filter(([_,v]) => v !== undefined)`, OR
> 3. Enable `ignoreUndefinedProperties: true` when initializing the admin Firestore instance
>
> We went with option 1/2 in `usage-log.ts` and option 1 everywhere in `lib/agent/memory/*`.

### 3. Firestore vector-with-filter needs a composite index

First implementation of `store.ts` did `.where("success","==",true).where("createdAt",">=",cutoff).findNearest(...)`. This forces a **composite vector index** that can't be created via the Firebase Console UI — requires `gcloud` CLI.

Fix: **drop the `where` clauses before `findNearest`**, overfetch 4× the requested k, filter in JavaScript. Only requires a simple single-field vector index on `embedding` (dimension 1024, COSINE), which Firebase CLI can deploy via `firestore.indexes.json`:

```json
{
  "collectionGroup": "episodes",
  "queryScope": "COLLECTION",
  "fields": [
    {
      "fieldPath": "embedding",
      "vectorConfig": { "dimension": 1024, "flat": {} }
    }
  ]
}
```

Deploy with `firebase deploy --only firestore:indexes`. Builds in seconds for an empty collection.

### 4. Shell backslash escaping eats one layer through single quotes

When writing the private key from a service account JSON into `.env.local` via a `node -e` one-liner, the Git Bash environment ate one layer of backslashes *despite* single quotes, so `replace(/\n/g, "\\n")` produced a single-char newline replacement instead of the literal `\n` sequence. The key ended up spread across ~30 lines of the env file, breaking `firebase-admin` parsing.

Fix: use **four** backslashes in the source (`"\\\\n"`) to produce the literal `\n` output, OR write the script to a file and run it via `node script.mjs` instead of `node -e`.

Prefer `scripts/*.mjs` files for anything non-trivial.

### 5. e5 query/passage prefix split produces ~0.97 not 1.0 for exact matches

The embedding model `intfloat/multilingual-e5-large-instruct` uses **different prefixes** for indexed passages (`passage: `) vs retrieval queries (`query: `). This produces *complementary* vectors, not identical ones. The smoke test's top-similarity score against the exact write query is ~0.9720, not 1.0000.

> [!info] This is correct
> **0.97 is the signature of proper prefix handling.** If you ever see 1.000 for a recall against the exact same text you wrote, that means the prefixes are missing or identical, and cross-document retrieval quality will suffer.

The prefix split is enforced in `embed.ts` via `embedQuery()` and `embedPassage()`. Don't bypass these by calling the Together endpoint directly.

---

## Smoke test script

`Cresento Website/Cresento.net/scripts/memory-smoke-test.mjs`

Run:
```bash
cd "Cresento Website/Cresento.net"
node scripts/memory-smoke-test.mjs
# or pass a specific orgId:
node scripts/memory-smoke-test.mjs <orgId>
```

What it does:
1. Loads `.env.local` manually (no dotenv dependency)
2. Initializes firebase-admin with the same shape as `lib/firebase-admin.ts`
3. Lists the 10 most recent episodes at `agentMemory/{orgId}/episodes/`
4. Counts sibling docs in `agentSessions` and `agentConversations`
5. Picks the most recent episode's `userQuery`, embeds it via Together (`query: ` prefix)
6. Runs `findNearest()` with k=5 on the `embedding` field
7. Reports similarity scores and whether the top match is the source episode

Verdict line at the end:
- `✅ PASS` if the source episode is the top result with similarity > 0.95
- `⚠️  Warning` if similarity is 0.7–0.95 (usually means prefix mismatch)
- `❌ FAIL` if similarity is below 0.7 or findNearest returned nothing

---

## 🛑 Hands off

- ❌ Don't call `embed.ts` directly from outside `lib/agent/memory/` — go through `getMemory()` so the agent never binds to Together AI specifically
- ❌ Don't write to `agentMemory/{orgId}/*`, `agentSessions/*`, or `agentConversations/*` from client code — server-side only (like `agentUsage`)
- ❌ Don't log raw embedding vectors — they're 1024 floats each, they bloat logs
- ❌ Don't switch embedding models without a migration plan — re-embedding the whole corpus is annoying
- ❌ Don't add `where()` filters before `findNearest()` in `store.ts` — that forces a composite vector index that can't be created via Firebase Console or Firebase CLI without more work. Keep filters in JS post-query and overfetch.
- ❌ Don't skip the `passage: ` / `query: ` prefix when embedding — e5 retrieval quality falls off sharply without the prefix split
- ❌ Don't use OpenRouter for embeddings or Together for chat completions. The provider split is intentional: OpenRouter for LLM, Together for embeddings only. Keeps rotation and rate-limiting independent.
- ❌ Don't spread objects with optional fields into Firestore writes — undefined values throw. Build payloads field-by-field or filter explicitly.
- ❌ Don't remove `export const maxDuration = 300` from `app/api/agent/route.ts` — the tool-use loop regularly runs 2–4 minutes for deep analyses, and the default Vercel timeout (10s on Hobby, 60s on Pro) will truncate them.

---

## Related

- [[Cresento Website]]
- [[Firebase Backend]] — the paging/wrapper rules this follows
- [[01 - Critical Preservation Rules]]
- [[Firestore Collection Audit 2026-04-11]] — current collection inventory
