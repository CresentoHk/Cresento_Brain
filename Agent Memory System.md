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

### 7. Recall context must NOT list prior tool names — it teaches mimicry without re-derivation

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

## Related

- [[Cresento Website]]
- [[Firebase Backend]] — the paging/wrapper rules this follows
- [[01 - Critical Preservation Rules]]
- [[Firestore Collection Audit 2026-04-11]] — current collection inventory
