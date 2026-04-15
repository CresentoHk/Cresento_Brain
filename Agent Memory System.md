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

> [!danger] ⚠️ HANDOFF WARNING FOR OTHER LLMs / AGENTS
> This system was shipped on 2026-04-11 in commit `74361a0` on branch `agent-demo`. Before editing ANY file in the manifest below, you MUST read [[01 - Critical Preservation Rules#🧠 Agent Memory System (Cora)]] and the [[#Gotchas discovered during Phase 1 bring-up]] section of this note. The design has 9 load-bearing rules that look like they could be "simplified" but will break silently if you change them. If in doubt, DO NOT edit — ask first.
>
> **Most common mistakes that will break this system:**
> 1. Adding `where()` filters before `findNearest()` in `store.ts` (forces composite vector index that can't be deployed from UI/CLI)
> 2. Removing `export const maxDuration = 300` from `app/api/agent/route.ts` (re-introduces the "fresh instance" timeout bug)
> 3. Calling Together AI for LLM tasks, or OpenRouter for embeddings (provider split is intentional)
> 4. Skipping the `passage: ` / `query: ` prefix in `embed.ts` (e5 retrieval quality collapses)
> 5. Spreading objects with optional fields into Firestore writes (`undefined` values throw)
> 6. Bypassing the `AgentMemory` interface by calling `embed.ts` or `FirestoreAgentMemory` directly
> 7. Changing the embedding model in `config.ts` without a migration plan (invalidates entire stored corpus)
> 8. **Making a heavy-data tool return raw payloads to the model.** All tools that load > ~5 KB of data must return a schema only (`aggregate`, `analytics`, `sandboxCatalog`, `analyzeHint`). The raw data stays in the sandbox. See gotcha #6 in this note.
> 9. **Converting `resume_needed` back into a user-facing error.** The auto-resume flow (server checkpoint → client re-POST → server resume from checkpoint) is how Cora recovers from empty iterations. Don't turn it into a spinner + error page.
> 10. **Removing the server heartbeat or the client's `heartbeat` no-op case.** Intermediate proxies kill idle SSE streams, and the heartbeat is what keeps them alive during silent model phases.
> 11. **Letting a non-string, non-array `content` field reach `messages[]` on a tool-result turn.** OpenRouter's Alibaba provider rejects the whole request with a 400 — `Invalid type for 'messages.[N].content': expected one of a string or array of objects, but got an object instead` — and the retry loop re-sends the same malformed messages forever. Every `role: "tool"` message MUST have `content: string` (JSON-stringify before assigning). Every assistant message with tool calls MUST have `content: "" | null | string | contentArray`. See gotcha #10.
> 12. **Overwriting `thinkingRoundsRef.current[last]` when a new tool call starts** (in `components/coach/agent-mode.tsx`). The `onToolCall` handler must APPEND a fresh empty round, not reset `currentThinkingRef.current` in place. Otherwise the coach only sees the final round's reasoning and every earlier round's thinking text vanishes. See gotcha #11.

## 📂 Touched files manifest

Full list of files that were created or modified in Phase 1. **Do not rewrite these without reading the rules first.** Line counts reflect the shipped commit `74361a0`.

### New files (`Cresento Website/Cresento.net/`)

| File | Lines | Purpose |
|---|---|---|
| `lib/agent/memory/config.ts` | 46 | Embedding model + thresholds |
| `lib/agent/memory/interface.ts` | 181 | `AgentMemory`, `Episode`, `Fact`, `Playbook`, `Conversation` types |
| `lib/agent/memory/embed.ts` | 126 | Together AI client (only place that calls Together) |
| `lib/agent/memory/store.ts` | 220 | `FirestoreAgentMemory` (recall + recordEpisode) |
| `lib/agent/memory/conversations.ts` | 117 | UI-facing transcript upsert |
| `lib/agent/memory/sessions.ts` | 275 | Mid-turn checkpoint/resume (fixes "fresh instance" bug) |
| `lib/agent/memory/index.ts` | 48 | `getMemory()` singleton + re-exports |
| `lib/agent/memory/README.md` | 428 | Integration guide + smoke test instructions |
| `scripts/memory-smoke-test.mjs` | 222 | End-to-end verification (safe to re-run, never prints secrets) |

### Modified files

| File | What changed |
|---|---|
| `app/api/agent/route.ts` | +203 lines. **9 integration edits.** `maxDuration = 300` + loadSession at top + recall injection + sessionId + fresh-or-resume messages builder + checkpointSession after each tool iteration + recordEpisode/upsertConversation/markSessionComplete in finally block. Capture `finalAssistantText` and `toolCallDigests` inside the stream. |
| `components/coach/agent-mode.tsx` | +26 lines. Added `sessionIdRef` (stable per conversation via `crypto.randomUUID()`), passed into `streamAgentResponse`, included in POST body. |
| `lib/agent/auth-guard.ts` | +8 lines. `AuthedUser` now returns `displayName` (from profile → decoded token → email → null). Memory records this as `coachName` for recall attribution. |
| `lib/agent/usage-log.ts` | +10 / -4. **Pre-existing bug fix unrelated to memory itself.** Was spreading `...event` into Firestore doc, hitting `Cannot use "undefined" as a Firestore value` on every successful run because `error?` was undefined. Now filters undefined fields. |
| `firestore.indexes.json` | **Not committed yet** — lives in working tree with user's parallel sessionMetrics indexes. Vector index is already deployed to the live Firestore DB via `firebase deploy --only firestore:indexes`. Will ship with user's next sessionMetrics commit. |

### New Firestore collections

| Path | Purpose | Writer | Reader |
|---|---|---|---|
| `agentMemory/{orgId}/episodes/{autoId}` | Q&A turns with 1024-dim embedding for recall | Server (admin SDK) | Server (admin SDK via `findNearest`) |
| `agentSessions/{sessionId}` | Mid-turn tool-loop state for resumption | Server | Server |
| `agentConversations/{sessionId}` | UI-facing transcript for recent-conversations sidebar | Server | Future: client (needs Firestore rule) |

### New env vars

| Name | Scope | Notes |
|---|---|---|
| `TOGETHER_API_KEY` | `.env.local` + Vercel | Only for embeddings. All LLM calls still go through `OPENROUTER_API_KEY`. |

### Firestore indexes

Single-field vector index on `episodes.embedding`:
- Dimension: 1024
- Distance: COSINE
- Deployed via `firebase deploy --only firestore:indexes`
- Defined in `firestore.indexes.json` (uncommitted, see above)

### Commit references

- **Website:** `CresentoHk/Cresento.net@74361a0` on branch `agent-demo` — `feat(agent): phase 1 long-term memory + session checkpoint resume`
- **Vault:** `CresentoHk/Cresento_Brain@15ce11d` on branch `main` — `docs(agent): phase 1 memory system live — smoke-tested + gotchas captured`

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

### 7. Chart renderings must emphasize the claim, not the raw data

> [!danger] Observed 2026-04-11
> Cora answered a "when should I sub Yonathan?" question and rendered three `speed_vs_fatigue` charts (one per game). Every chart:
> - Had identical title `"Speed Trend & Stamina Decay — Yonathan Bensadon"` — the coach couldn't tell which game was which
> - Showed raw 20 Hz speed as a noisy line with a linear-regression trend slope of ~0.14–0.25 km/h per hour — visually indistinguishable from flat on a 0–25 km/h axis
> - The `distance_per_minute` charts colored bars green/gray based on above/below session average, but the legend was disabled so the color split looked arbitrary

**Architectural rules for chart renderings:**

1. **Per-session charts must include the session date in the title.** `loadSessionForChart()` (in `lib/agent/tools.ts`) returns a `dateLabel` field derived from `startTime → uploadedAt → first speed sample`. Every per-session chart renderer must accept an optional `sessionLabel` param and append `— ${sessionLabel}` to the title. Without it, multi-game answers show a wall of identical titles.

2. **Charts must emphasize the conclusion the model is drawing**, not the raw data:
   - Replace raw noisy time series with a **rolling average** as the primary signal (3-min rolling average in `renderSpeedWithRegression`). Raw data becomes a dim background reference (opacity 0.18).
   - Draw **reference lines** for the specific scalar values the model is reasoning about. For fatigue: a horizontal line at the baseline-window average and another at the end-window average. The visible gap between them IS the fatigue claim.
   - Subtitle the chart title with the delta: `"Decay: 9.1 → 7.4 km/h (-18.7%)"`. The viewer should be able to tell whether the player is fatigued without reading the model's prose.

3. **Any color differentiation needs a visible explanation.** If a chart colors some bars one way and others another way, the legend must contain swatches labeled with what the colors mean. Use **legend-only proxy datasets** (data array of nulls, colored swatches) in Chart.js — they populate the legend without adding visible bars. Example in `renderDistancePerMinute`: "Above session avg (N m)" + "Below session avg".

4. **Mix bar + line datasets** using Chart.js's `type: "line"` dataset inside a `type: "bar"` chart config. Works on QuickChart without extra plugins. Use this to draw horizontal reference lines (averages, thresholds, targets) on top of bar charts without needing the annotation plugin.

5. **Do NOT use Chart.js's `chartjs-plugin-annotation`.** QuickChart supports it but config is verbose and error-prone. Mixed datasets are simpler and render identically.

6. **Color palette for fatigue/performance charts:**
   - Foreground signal (rolling average, above-average bars): `#10b981` (emerald-500)
   - Background / below-average reference: `#4b5563` (slate-600) or rgba(8, 146, 118, 0.18)
   - Reference line for baseline: `#60a5fa` (blue-400, dashed)
   - Reference line for end window / session average: `#f59e0b` (amber-500, dashed)
   - These are used in `renderSpeedWithRegression` and `renderDistancePerMinute` — keep other chart renderers visually consistent.

**Phase C implementation**, commit `2a58f32`:
- `loadSessionForChart` now returns `{data, name, dateLabel}`
- All 8 per-session cases in `execRenderChart` pass the date into renderers and append it to `chartTitle`
- `renderSpeedWithRegression` rewritten to show rolling avg + baseline/end reference lines + delta subtitle
- `renderDistancePerMinute` now has a legend explaining the color split and a mixed bar+line dataset with the session average

### 6. Tool results must NEVER flow through the chat context — schema-first architecture

> [!danger] Cora's "stuck mid-analysis" bug, observed 2026-04-11
> Coach asked a 3-game fatigue-inflection question. Cora called `get_session_timeseries` three times successfully, then went silent for minutes during the reasoning phase before `analyze_with_code`. The SSE stream emitted no visible events. User saw an infinite spinner. Root cause: each `get_session_timeseries` call was returning 30–60 KB of data (downsampled speed points + full precomputedAnalytics blob + aggregate stats) into the `messages[]` array. Three calls × 60 KB = 180 KB of tool-result content in context on top of the system prompt and reasoning. qwen3.6-plus couldn't hold it together and reasoned slowly / got stuck.

**Architectural rule, to be applied to every heavy-data tool:**

```
Tools that load large payloads must NOT return that data to the model.
Instead they return a SCHEMA:
  - Top-line scalar aggregates (small — totalDistance, maxSpeed, ...)
  - Key precomputed scalars (score, ratio, slope, r²)
  - A sandbox catalog: field paths + types describing what the data
    looks like INSIDE the analyze_with_code VM
  - An analyzeHint with an example tool call pre-populated

The actual raw data NEVER enters the `messages[]` context. It lives
only inside the sandbox, loaded fresh by analyze_with_code from the
canonical Firestore collection per-call.
```

**Why this works without a persistent workspace:**
- `analyze_with_code` in `lib/agent/tools.ts` already calls `loadSessionForSandbox(sessionId, orgId, ...)` itself on every invocation
- The model doesn't need to "transfer" data from `get_session_timeseries` to `analyze_with_code` — both tools reach the same Firestore path
- The schema tool tells the model *what exists* and *what field paths to use* in the sandbox; the compute tool loads the data at the moment it's needed
- Zero cross-call state, zero new infrastructure

**Phase B implementation, commit `69b4670`:**
- `get_session_timeseries` (lib/agent/tools.ts) now returns `{metadata, aggregate, analytics, sandboxCatalog, analyzeHint}` — ~1.5 KB instead of 30–60 KB
- `AGENT_TOOLS` schema in tools.ts rewritten: the `description` explicitly tells the model "this does NOT return raw data — it returns a schema; use analyze_with_code to compute"
- System prompt in `lib/agent/config.ts` has a new "Step A: Discovery / Step B: Computation" section teaching the two-step pattern. Recipes updated.
- Dropped `fields` and `downsample_to` params (they're meaningless now that no raw data is returned). Added optional `alias` param so the sandbox catalog shows the field paths the model will actually type into its code.

**Not yet converted (deliberately, for scope):**
- `query_game_metrics`, `query_session_metrics`, `query_metric_summary`, `query_player_insights`, `query_team_overview` — these return smaller category-based payloads and the benefit is smaller. Migrate incrementally if their payloads become a problem. When you do, use the same pattern: `{aggregate, analytics, sandboxCatalog, analyzeHint}`.

**Rules for future heavy-data tools:**
- ❌ Do not return arrays with > ~50 items to the model
- ❌ Do not return "raw" anything — by the time it reaches the model, it's already schema
- ❌ Do not add `downsample_to` or `fields` params as escape hatches — if the model needs the data, the sandbox needs it
- ✅ Return scalar aggregates (totals, maxes, ratios, scores) — the model needs these to reason
- ✅ Return a `sandboxCatalog` with field paths matching `loadSessionForSandbox()` exactly
- ✅ Return an `analyzeHint` with an example `analyze_with_code` call pre-populated

### 8. segmentedFatigueScore — canonical field now in `sessionMetrics`

> [!info] Option B steps 1–3 shipped 2026-04-12. Steps 4–5 remain.
> The raw `segmentedFatigue.score` stored in Firestore is in the LEGACY direction (higher = LESS fatigued). The canonical field `sessionMetrics/{id}.segmentedFatigueScore0to10` stores the flipped value (higher = MORE fatigued, matching all coach-facing labels).
>
> **What shipped (Option B steps 1–3):**
> - ✅ Cloud Function (`functions/src/index.ts`) writes `segmentedFatigueScore0to10` after `calculateSegmentedFatigue` runs
> - ✅ Backfill script (`scripts/backfill-sessionMetrics.mjs`) already handles the flip — ready to run with `--yes`
> - ✅ Agent tools read via `getCanonicalFatigueScore(sessionId, rawFallback)`: tries `sessionMetrics` first, falls back to `flipSegmentedFatigue()` for unbackfilled sessions
> - ✅ `metric-docs.ts` updated to reference the canonical field
>
> **Rule for new Agent Mode reads:** use `getCanonicalFatigueScore()` (async). For bulk/sync paths (like SUMMARY_STAT_MAP), `flipSegmentedFatigue()` is still acceptable.
>
> **Inside the analyze_with_code sandbox**, `<alias>.precomputed.segmentedFatigue.score` is STILL the raw value. Code that reads it must apply `10 - score` manually.
>
> **What remains (steps 4–5):** Website UI migration + removal of `flipSegmentedFatigue`. See the TODO section below.

### 9. Recall context must NOT list prior tool names — it teaches mimicry without re-derivation

> [!danger] Observed in real use on 2026-04-11
> The first version of the recall injection included a `"Tools used: plan_analysis, query_player_insights, ..."` line under each recalled episode. When a coach re-asked a semantically similar question, the model copied the tool sequence from the recall block — **but skipped the argument-derivation steps the original run had done.** In the reported incident, the model called `query_player_insights` and `query_metric_summary` with `player_id: "Yonathan Bensadon"` (the player's **name**, not their Firestore ID) because the recall summary didn't show the `get_team_roster → find ID` step that preceded those calls the first time.
>
> Worse, when the model tried to recover from the failed tool calls, qwen3.6-plus **fell out of native function calling** and started emitting text-format tool calls (`<tool_call><function=get_team_roster>...</function></tool_call>`), which the server has no parser for. The stream closed on the text as the "final answer" and the user saw broken output.

Fixes applied in route.ts:
- **Remove the "Tools used" line entirely** from the recall context. Prior tool names are informational noise that encourages copy-paste without re-derivation.
- **Shorten the prior answer snippet** from 500 → 300 chars and rename the header to "Past conclusion" so the model treats it as breadcrumb context, not an authoritative answer to paraphrase.
- **Reduce `k` from 5 → 3** — fewer prior episodes in context means less pressure on the system-prompt length budget that pushes qwen into text-mode tool calls.
- **Rewrite the usage instruction** from "You MAY reuse the tool approach" to "Use this ONLY to understand what the coach cares about — do not copy numbers, player IDs, or tool arguments from these snippets. Re-derive every fact via fresh tool calls."
- **Add a text-mode detector** in the final-response branch of the tool loop that logs a warning whenever `<tool_call>` or `<function=` appears in the streamed content, so we can tell if this recurs.

> [!info] Why qwen falls out of native function calling
> The trigger is long or noisy system prompts combined with any assistant/context text that looks like a tool call. The model briefly "forgets" whether it's supposed to use native OpenAI-shape function calling or Qwen's older text format, and emits the wrong one. Keeping the recall context terse and tool-name-free is the main defense.

### 10. OpenRouter 400 — `messages[N].content` object rejected by Alibaba provider

> [!danger] Observed 2026-04-11 during a 3-game "when should I sub Yonathan" run
> Cora successfully ran `plan_analysis` → `browse_calendar` + `get_team_roster` → 3× `load_game_data` → 3× `get_session_timeseries` + `describe_metrics` → `analyze_with_code` → `render_chart` (×2 failed `speed_timeline`) → `browse_charts` → `render_chart` (×3 `speed_vs_fatigue`) → 2 more `analyze_with_code` rounds → and produced a full prose answer with a Markdown table and the recommendation "Sub at Minute 75–80". **38 tool calls total.** Then the stream died mid-render with:
>
> ```json
> {"error":{"message":"Provider returned error","code":400,
>   "metadata":{
>     "raw":"{\"error\":{\"message\":\"Invalid type for 'messages.[19].content': expected one of a string or array of objects, but got an object instead.\",\"type\":\"invalid_request_error\",\"param\":\"'messages.[19].content'\",\"code\":\"invalid_type\"}}",
>     "provider_name":"Alibaba"
>   }
> }}
> ```
>
> Alibaba's OpenAI-compatible endpoint is strict: every `content` field must be `string | Array<{type:..., ...}> | null`. A raw JS object (e.g. `{aggregate:..., analytics:...}`) is rejected with a permanent 400. Worse, the existing auto-resume loop **re-sent the same malformed `messages[]` three more times** before giving up, burning tokens and delaying the error surface to the coach.

**Root cause hypotheses — all need investigation:**

1. **`executeTool()` return value shape.** Every tool handler in `lib/agent/tools.ts` must return `content: string` (JSON-stringified). Some branch — likely `execGetSessionTimeseries`, `execAnalyzeWithCode`, or a chart renderer error path — is returning an object literal instead of a JSON-stringified string. Grep `execXxx` functions for any `return { ... }` that isn't wrapped in `JSON.stringify()` before assignment to `content`.
2. **`sanitizeForCheckpoint()` fall-through.** In `lib/agent/memory/sessions.ts`, the sanitizer handles `string`, `Array`, and `null` content explicitly but falls through to `"[unsupported content type stripped]"` for objects. If the sanitizer is skipped on a path — e.g. `appendToolResult()` writes directly to the in-memory `messages[]` before the next checkpoint round — an object slips through and the next OpenRouter call fails.
3. **Chart image result merge.** `render_chart` returns a vision content array `[{type:"text",...}, {type:"image_url", image_url:{url:...}}]`. If on resume the server reconstructs `messages[]` from the checkpoint and merges the array with some server-side annotations, the result could become an object (`{type:"text", ...}` without the outer array). Check the resume path in `app/api/agent/route.ts` around `loadSession()` where messages are re-hydrated.
4. **`analyze_with_code` result shape.** The sandbox returns `{result, logs, error?, elapsedMs}` — if this is passed directly to `content` anywhere instead of being `JSON.stringify()`'d, it'll be an object at index N. Verify `execAnalyzeWithCode` returns `content: JSON.stringify({...})` at every return site including error paths.

**Defense rule going forward — enforce at a single chokepoint:**

```typescript
// In route.ts, before appending ANY message from a tool result to messages[]:
function coerceToolContent(raw: unknown): string | ContentBlock[] {
  if (typeof raw === "string") return raw
  if (Array.isArray(raw)) return raw  // vision content blocks
  if (raw === null || raw === undefined) return ""
  // Defensive: never let an object reach OpenRouter as content
  try { return JSON.stringify(raw) } catch { return String(raw) }
}
```

Every `messages.push({role: "tool", tool_call_id, content: coerceToolContent(...)})` should flow through this helper. **Do not rely on each tool handler returning the right shape** — centralize the guard.

**Log the offender.** Before `coerceToolContent` falls through to the object branch, `console.warn("[Agent] Tool result was object — coerced:", { toolName, keys: Object.keys(raw as object) })`. Next reproduction tells us which tool is the culprit.

### 11. Thinking rounds clobber each other — only the final round is visible to the coach

> [!danger] Observed 2026-04-11
> Coach said: "im also unable to see all the thinking in between tools". Confirmed in code: in `components/coach/agent-mode.tsx` lines ~867–903 the streaming handlers work like this:
>
> - `onThinking(token)` — appends to `currentThinkingRef.current`, then at line 872–880 it either pushes a first round or **overwrites `rounds[last]`** with `currentThinkingRef.current`.
> - `onToolCall(name, args)` — resets `currentThinkingRef.current = ""` at line 899, then does `thinkingRoundsRef.current = [...thinkingRoundsRef.current]` (just a shallow copy) at line 902. **It never pushes a new empty round.**
>
> Result: when the next `onThinking` token arrives after a tool call, `currentThinkingRef.current` starts from `""`, and the check `rounds[last] !== currentThinkingRef.current` is true, so the code falls into the `rounds[rounds.length - 1] = currentThinkingRef.current` branch — **overwriting the previous round's completed text with the new round's starting text.** Every tool boundary silently eats the previous round.
>
> User sees only the LAST reasoning round, which is often a short "synthesize the final answer" blurb. The in-between reasoning — the part that explains why Cora chose `analyze_with_code` over `query_metric_summary`, why she picked 3-minute rolling windows, why she excluded the first 5 minutes — is invisible.

**Fix (pending):**

In `onToolCall`, after detecting a tool boundary, explicitly push a new empty round:

```typescript
// onToolCall — finalize current thinking round, start a new one
(name, args) => {
  // If there's in-flight thinking text, leave it in place as a completed round
  // and START A NEW ROUND so the next onThinking tokens append, not overwrite.
  if (currentThinkingRef.current.trim().length > 0) {
    thinkingRoundsRef.current = [...thinkingRoundsRef.current, ""]
  }
  currentThinkingRef.current = ""
  // ... rest of the handler unchanged
}
```

And in `onThinking`, the check at line 873 must be: *if the last round is the "active" empty-started round, append; otherwise push a new round*. Safer rewrite:

```typescript
(token) => {
  currentThinkingRef.current += token
  setStatusLabel("Reasoning...")
  const rounds = [...thinkingRoundsRef.current]
  if (rounds.length === 0) {
    rounds.push(currentThinkingRef.current)
  } else {
    // The last round IS the currently-streaming round by contract
    rounds[rounds.length - 1] = currentThinkingRef.current
  }
  thinkingRoundsRef.current = rounds
  // ... setMessages unchanged
}
```

The contract becomes: **`thinkingRoundsRef.current[last]` is always the currently-streaming round. `onToolCall` is the ONLY place that closes a round and opens a new one, by pushing `""` onto the array.**

`ThinkingRounds` component at line 1265 already renders every round in the array, so once the ref contains all rounds they'll all display. No UI change needed — this is a pure state-management bug in the stream handlers.

### 6. e5 query/passage prefix split produces ~0.97 not 1.0 for exact matches

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

## TODO: Option B — canonical segmentedFatigue migration

> [!todo] Steps 1–3 DONE (2026-04-12). Steps 4–5 remain.
> **Status:** Option B steps 1–3 shipped on 2026-04-12. The Cloud Function now writes `segmentedFatigueScore0to10` to `sessionMetrics/{id}`. Agent tools read it via `getCanonicalFatigueScore()`. **Full backfill completed 2026-04-12** — all 1,425 `sessionMetrics` docs exist with 80+ fields (not just fatigue). Stats: 844 from cloud_function, 581 from backfill, 0 from phone. 774 have distance data, 990 have maxSpeed, 108 have segmented fatigue, 56 have ACWR, 948 have regression. Non-Agent-Mode surfaces (website calendar, dashboard) still read from `sensorData` — that's steps 4–5 (Phase D of the migration plan).

**The plan**, from [[SessionMetrics Migration Plan#Phase B — Update the Cloud Function to write cross-session metrics to `sessionMetrics`]]:

### Step 1 — Cloud Function writes flipped score to sessionMetrics

File: `Cresento Website/Cresento.net/functions/src/index.ts`

In `processAndSaveAnalytics()` after `calculateSegmentedFatigue()` runs, also write to `sessionMetrics/{sessionId}`:

```typescript
const flippedSegScore =
  segmentedFatigue && typeof segmentedFatigue.score === "number"
    ? Math.max(0, Math.min(10, 10 - segmentedFatigue.score))
    : null
const flippedConfidence = segmentedFatigue?.confidence ?? null

await db.collection("sessionMetrics").doc(sessionId).set(
  {
    segmentedFatigueScore0to10: flippedSegScore,
    segmentedFatigueConfidence: flippedConfidence,
    metricsLastSource: "cloud_function",
    metricsComputedAt: admin.firestore.FieldValue.serverTimestamp(),
  },
  { merge: true }
)
```

Use `set({merge: true})` so the phone-written fields from Phase A (already shipped in RN app's `SessionMetricsBuilder.ts`) are preserved.

### Step 2 — Backfill historical sessions

File: `Cresento Website/Cresento.net/scripts/backfill-sessionMetrics.mjs`

Walk every existing `sensorData/{id}` doc, extract `precomputedAnalytics.segmentedFatigue.score` and `.confidence`, flip, write to `sessionMetrics/{id}`. Idempotent — running twice produces the same result.

**Run procedure:**
1. `node scripts/backfill-sessionMetrics.mjs --dry-run` — prints count of docs touched and a sample of flipped values
2. Review the dry-run output. If a flipped value looks wrong for a known session, abort.
3. `node scripts/backfill-sessionMetrics.mjs --yes` — applies in batches of 400 with a 500 ms delay between batches

### Step 3 — Agent Mode reads prefer new field, fall back to flip

File: `Cresento Website/Cresento.net/lib/agent/tools.ts`

At each of the 5 current `flipSegmentedFatigue()` call sites, change the read to:

```typescript
// Prefer the canonical flipped field. Fall back to flipping the raw
// stored score on the fly for sessions that haven't been backfilled yet.
const sm = await getSessionMetrics(sessionId) // new helper in firestore.ts
const flippedScore =
  sm?.segmentedFatigueScore0to10 ??
  flipSegmentedFatigue(analytics?.segmentedFatigue?.score)
```

Keep `flipSegmentedFatigue()` alive during the rollout. Delete it once every session has been backfilled and every new session is being written via Step 1.

### Step 4 — Website UI migration

Files: `lib/firestore.ts`, `components/dashboard/sessions-calendar.tsx`, `components/analytics/*`, `app/games/[id]/analytics/page.tsx`, `lib/utils.ts`

Add a `getSessionMetrics(sessionId)` helper in `firestore.ts` that reads from `sessionMetrics/{sessionId}` with a fallback to flipping the raw score. Migrate the calendar and dashboard components one at a time — this is Phase D in the full migration plan and touches the ~70 KB `sessions-calendar.tsx` critical file, so do it under a feature flag and verify each surface before removing the fallback.

### Step 5 — Remove `flipSegmentedFatigue` and this gotcha

Once Steps 1–4 are done and monitored for a week:
- Delete `flipSegmentedFatigue()` from `lib/agent/tools.ts`
- Delete the ⚠️ warning from the sandboxCatalog entry in `execGetSessionTimeseries`
- Update [[01 - Critical Preservation Rules#🧮 StatsEngine consistency across platforms]] to mark the label bug as fixed
- Remove gotcha #8 from this note (the read-time flip rule becomes dead)

### Risks to watch

- **Cloud Function deploys restart live infrastructure.** Deploy during low-traffic hours.
- **The backfill script writes to production Firestore.** Always dry-run first. Check one known session's flipped value matches expectations before applying to all ~thousands.
- **`sessions-calendar.tsx` is a critical 70 KB file.** Migration under a feature flag. If a coach complains about a fatigue number, the flag is the rollback.
- **The client-side `lib/utils.ts` formula** still outputs the raw direction. It's currently only called from a few non-Agent paths. Decide whether to rewrite the formula to output the flipped direction directly, or keep flipping in the caller — the latter is safer but duplicates the flip logic.

### Blast radius

| Surface | Current (Option A) | After Option B |
|---|---|---|
| Agent Mode text output | ✅ Flipped | ✅ Reads `segmentedFatigueScore0to10` directly |
| Agent Mode charts | ✅ Flipped | ✅ Reads new field |
| Agent Mode sandbox (analyze_with_code) | ⚠️ Raw in sandbox, warning in catalog | ✅ New field also in catalog, raw still available |
| Website sessions calendar | ❌ Raw (wrong) | ✅ Reads new field |
| Website dashboard | ❌ Raw (wrong) | ✅ Reads new field |
| Website analytics pages | ❌ Raw (wrong) | ✅ Reads new field |
| iOS app | N/A (no segmented impl) | N/A |
| RN app | N/A (writes phone-only fields, leaves segmented null for Cloud Function) | ✅ Will auto-read whatever field is written |

---

## TODO: Robust retry classification

> [!todo] Current `runWithResume` auto-retry is too blunt — it treats every failure as retryable
> When Cora hit the `messages[19].content` 400 on 2026-04-11, the client-side `runWithResume` loop in `components/coach/agent-mode.tsx` re-POSTed the same malformed conversation state three times before giving up (`MAX_AUTO_RESUMES = 3`). All three retries failed with the same permanent 400 because the malformed message was now checkpointed into `agentSessions/{sessionId}` and being reloaded on resume. This wasted ~30 s of coach time and ~15k tokens, and Cora's final answer (which was already fully written before the stream died) was lost.
>
> Related: the current retry logic only triggers on `resume_needed` (empty-iteration). It does NOT currently trigger on upstream 5xx or timeouts — those just surface as errors. So we have the opposite problem too: transient failures get no retry while permanent failures get three.

**Failure taxonomy — what retry strategy each class needs:**

| Class | Example | Retry? | Strategy |
|---|---|---|---|
| Empty iteration | `content === "" && tool_calls.length === 0` | ✅ Yes | Checkpoint, `resume_needed`, client re-POST. **Already implemented.** |
| OpenRouter 5xx | `502 Bad Gateway`, `503 Service Unavailable` | ✅ Yes | Exponential backoff (1s → 3s → 8s), max 3 tries. **Not implemented.** |
| OpenRouter read timeout | 90 s `AbortController` fires | ✅ Yes | Single retry after 2s — might be a cold provider route. **Not implemented.** |
| OpenRouter 429 rate limit | `429 Too Many Requests` | ✅ Yes | Respect `Retry-After` header. **Not implemented.** |
| OpenRouter 400 malformed | `messages.[N].content` wrong type | ⚠️ **Self-repair then retry once** | Scan `messages[]`, coerce any object `content` fields to strings via `coerceToolContent()`, retry with sanitized array. If second attempt also fails, surface as permanent. **Not implemented.** |
| OpenRouter 400 other | Invalid tool schema, unknown param | ❌ No | Permanent — surface immediately with the provider error message. **Not implemented.** |
| OpenRouter 401 / 403 | Bad API key | ❌ No | Permanent — surface with a clear "check `OPENROUTER_API_KEY`" message. **Not implemented.** |
| Provider downstream 400 | Alibaba/Groq/Fireworks-specific rejection | ⚠️ Failover | Retry once with OpenRouter's `provider.order` set to exclude the failing one (`provider: {order: [...], allow_fallbacks: true}`). **Not implemented.** |
| Stream closed unexpectedly | Network flap | ✅ Yes | Checkpoint + resume. **Already implemented via empty-iteration path, but only by accident.** |

**What to build:**

1. **Server-side classifier in `route.ts`.** Wrap every `streamCallOpenRouter` call in a `try/catch` and emit a new SSE event type `retry_decision` with `{class, willRetry, attemptsRemaining, waitMs, reason}`. The client uses this to either silently retry, surface the error, or show a "retrying in 3s…" status label.

2. **Permanent-error SSE event.** Add `emit({type: "error_permanent", message, class})` to the server. Client handles this by stopping the auto-resume loop immediately and showing the error in the chat transcript — no "Cora couldn't finish after multiple retries" wrapper.

3. **Self-repair path for `content` type errors.** Before the 400 retry, walk `messages[]` and coerce any `content` field where `typeof === "object" && !Array.isArray()` to `JSON.stringify(content)`. Log which message index was repaired. If the retry succeeds, we have the evidence of which tool's handler is the culprit and can fix the root cause.

4. **`coerceToolContent()` helper at the single chokepoint** — see gotcha #10. This prevents the 400 from happening in the first place, but the self-repair path above is the backstop for when a new tool gets added and the author forgets.

5. **Checkpoint scrubbing on resume.** In `loadSession()` inside `lib/agent/memory/sessions.ts`, re-run the same `coerceToolContent` scan on every message before handing them back to the tool loop. If a bad message slipped past the original write, the resume path can't propagate it.

6. **Client-side: on `error_permanent`, do NOT increment `resumeCountRef`.** Currently every retry attempt — permanent or not — eats one of the three slots. Permanent errors should short-circuit to `runWithResume` returning immediately with the error visible.

**Files that will change:**

- `app/api/agent/route.ts` — add classifier, new SSE event types, `coerceToolContent` chokepoint around `messages.push({role: "tool", ...})`
- `components/coach/agent-mode.tsx` — handle `retry_decision` (show status label) and `error_permanent` (break the resume loop, show error)
- `lib/agent/memory/sessions.ts` — scrub content on load
- `lib/agent/tools.ts` — audit every `execXxx` return site, ensure `content` is always a string (JSON-stringified for objects)

**Smoke test after implementation:**

- Force a 400 by injecting `{content: {foo: "bar"}}` into a tool result in dev. Confirm: server classifies as `malformed_content`, emits `retry_decision` with self-repair, retry succeeds, debug log shows the repaired message index.
- Force a 5xx by pointing `OPENROUTER_BASE_URL` at `https://httpstat.us/502`. Confirm: exponential backoff, retries succeed when URL is restored.
- Force an auth failure by blanking `OPENROUTER_API_KEY`. Confirm: immediate `error_permanent`, no retries, clear message.

---

## TODO: Thinking visibility — stop clobbering rounds at tool boundaries

> [!todo] Coach can't see reasoning between tool calls (reported 2026-04-11)
> Root cause is documented in gotcha #11 above. The fix is small — a single-file change in `components/coach/agent-mode.tsx` around lines 867–903 — but it needs to be shipped with a manual QA pass because the `ThinkingRounds` component rendering logic is already wired for the multi-round case.

**Files that will change:**

- `components/coach/agent-mode.tsx` — rewrite `onToolCall` and `onThinking` handlers per gotcha #11. That's it.

**What the fix is:**

1. Establish the contract: `thinkingRoundsRef.current[last]` is ALWAYS the currently-streaming round. Non-last entries are completed rounds and never mutated.
2. `onThinking(token)`: append token to `currentThinkingRef.current`, then `rounds[last] = currentThinkingRef.current`. If `rounds.length === 0`, push first.
3. `onToolCall(name, args)`: if `currentThinkingRef.current.trim().length > 0`, push a new empty `""` onto `thinkingRoundsRef.current` — this creates the new active round. Reset `currentThinkingRef.current = ""`.
4. `onToolResult(name, result)`: leave `thinkingRoundsRef.current` alone. The new round is the empty string pushed in step 3; the next `onThinking` tokens will populate it.
5. Final pruning in the `done` handler (line ~1019) already filters empty rounds: `const finalRounds = thinkingRoundsRef.current.filter(r => r.trim().length > 0)`. That stays.

**Manual QA after fix:**

- Ask Cora a multi-tool question (e.g. "When should I sub Yonathan?"). Expand the thinking panel after the response arrives.
- Confirm: 3+ distinct rounds visible, each with its own pre-tool-call reasoning. Round 1 should be the `plan_analysis` justification, round 2 the `get_session_timeseries`/`describe_metrics` setup, round 3 the `analyze_with_code` setup, etc.
- Confirm: the final round (the one before answering) contains the "synthesize the answer" reasoning.
- Confirm: no round is a repeat or prefix of another — that would mean the clobber bug is still present.

**Related:** the server already emits separate `thinking` SSE events between each tool call (verified in `app/api/agent/route.ts` reasoning-token branch). The server is fine. This is purely a client state-management bug.

---

## Related

- [[Cresento Website]]
- [[Firebase Backend]] — the paging/wrapper rules this follows
- [[01 - Critical Preservation Rules]]
- [[Firestore Collection Audit 2026-04-11]] — current collection inventory
- [[SessionMetrics Migration Plan]] — Phase B above is Phase B there
