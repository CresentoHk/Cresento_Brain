---
title: Agent Memory System
type: shared-system
tags:
  - project/website
  - backend
  - agent
  - ml
created: 2026-04-11
status: phase-1-applied
location: "Cresento Website/Cresento.net/lib/agent/memory/"
---

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
| `lib/agent/memory/config.ts`               | ✅ Written                                 |
| `lib/agent/memory/interface.ts`            | ✅ Written                                 |
| `lib/agent/memory/embed.ts`                | ✅ Written                                 |
| `lib/agent/memory/store.ts`                | ✅ Written                                 |
| `lib/agent/memory/conversations.ts`        | ✅ Written                                 |
| `lib/agent/memory/sessions.ts`             | ✅ Written                                 |
| `lib/agent/memory/index.ts`                | ✅ Written                                 |
| `lib/agent/memory/README.md` (integration) | ✅ Written                                 |
| `lib/agent/auth-guard.ts` exposes displayName | ✅ Applied                               |
| `app/api/agent/route.ts` edits (9 edits)   | ✅ Applied                                 |
| `export const maxDuration = 300` in route  | ✅ Applied                                 |
| `components/coach/agent-mode.tsx` sessionId | ✅ Applied                                |
| `TOGETHER_API_KEY` in `.env.local`         | ⏳ Manual add required                     |
| `TOGETHER_API_KEY` on Vercel               | ⏳ Manual add required                     |
| Firestore vector index                     | ⏳ Auto-prompted on first run              |
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

## 🛑 Hands off

- ❌ Don't call `embed.ts` directly — go through `AgentMemory` so the agent never binds to Together AI specifically
- ❌ Don't write to `agentMemory/{orgId}/...` from client code — server-side only (like `agentUsage`)
- ❌ Don't log raw embedding vectors — they're 1024 floats each, they bloat logs
- ❌ Don't switch embedding models without a migration plan — re-embedding the whole corpus is annoying

---

## Related

- [[Cresento Website]]
- [[Firebase Backend]] — the paging/wrapper rules this follows
- [[01 - Critical Preservation Rules]]
- [[Firestore Collection Audit 2026-04-11]] — current collection inventory
