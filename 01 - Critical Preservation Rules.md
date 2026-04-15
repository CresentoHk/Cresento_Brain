---
title: Critical Preservation Rules
type: rules
tags:
  - critical
  - shared
  - protocol
created: 2026-04-11
aliases:
  - Do Not Break
  - Preservation Rules
---

# 🛑 Critical Preservation Rules

> [!danger] Rule Zero
> **Do not break existing functionality.** Read the existing code BEFORE editing. When in doubt, **add**, do not modify. Preserve all callers.

This is the master "do not touch" list. Every item here exists because something downstream will silently break if you change it. Each rule links to the project notes that depend on it.

---

## 🔵 BLE protocol — frozen wire format

> [!danger] Never change these
> The same protocol is implemented FOUR times (Nordic firmware, ESP firmware, RN BleContext, iOS BLEViewModel). Changing one side without the others = bricked devices in the field.

- **Service UUID:** `12345678-1234-1234-1234-1234567890AB`
- **8 Characteristics** (Control, Data, BattState, BattPct, BattVolt, ChargingEvt, ChargingTime, DeviceID)
- **Command bytes** (start log, stop log, flush, get battery, etc.)
- **Notification packet format** (delta-pack compressed IMU, optional gzip, sequence numbers)
- **Zombie connection detection** — 5-second timeout is intentional, do not "optimise" it

See: [[BLE Protocol]], [[Nordic BLE Firmware (nRF52811)]], [[ESP32-S3 SmallBoard Firmware]]

---

## 💡 LED sequences — timing-sensitive

> [!warning] Test on hardware before merging
> LED behaviour communicates state to the user with no other UI. Off-by-one timings here look like "the chip is broken" to a coach in the field.

- LED pin assignments in Nordic firmware: P0.13 (red), P0.08 (green), P0.09 (blue)
- ESP board uses Adafruit NeoPixel — different driver, similar semantics
- **Always explain WHY a proposed LED change is safe** before making it
- The user has explicitly called this out as the most fragile part of the chip code

See: [[Nordic BLE Firmware (nRF52811)#LED behaviour]], [[ESP32-S3 SmallBoard Firmware#LED behaviour]]

---

## 🔁 Session state machine

> [!danger] Carefully tuned — do not restructure
> States: **idle → logging → flushing → idle**

- The flushing state exists to drain the W25Q128 flash before the chip is allowed back to idle
- Mid-flush disconnects must resume on reconnect, not restart
- Charging events transition state too — see [[Session State Machine]]

Touching any side of this (firmware OR app) will cause silent data loss.

---

## 📦 Firestore session paging

> [!warning] 700 KB page cap is load-bearing
> Firestore documents have a hard 1 MB limit. The 700 KB cap leaves headroom for metadata + atomic batch overhead. **Never bypass `SessionUploader`** to write `sensorData` directly.

- Paging logic lives in `Cresento/src/src/utils/SessionUploader.ts`
- All three apps (RN, iOS, Web) read paged sessions — changing the page schema breaks ALL clients
- Use existing wrappers: `lib/firestore.ts` on web, `SessionUploader.ts` + utils on RN, ViewModels on iOS

See: [[Firebase Backend]], [[Cresento React Native App#SessionUploader]]

---

## 🗓️ Calendar view session loader

> [!warning] Critical user-facing path
> The website's `components/dashboard/sessions-calendar.tsx` (~70 KB) loads sessions for the calendar. Changes to how sessions are queried, paged, or grouped here directly impact what coaches see.

- Pairs with `lib/firestore.ts` and `lib/analytics/`
- Trim functionality lives in `lib/trim-utils.ts` (~31 KB) — also must be preserved

See: [[Cresento Website#Sessions calendar]]

---

## 🧮 StatsEngine consistency across platforms

> [!info] Three implementations, one truth
> - RN: `Cresento/src/src/utils/StatsEngine.ts` (~37 KB)
> - Web: `Cresento Website/Cresento.net/lib/analytics/`
> - iOS: `DataRecoveryIOS/.../StatsEngine.swift`
>
> All three must produce **the same numbers** for the same input session. If you change a metric formula, change it in all three or you'll get reports of "the website says X but the app says Y".

See: [[StatsEngine Cross-Platform]]

> [!info] Fixed (Agent Mode) 2026-04-12 via Option B — canonical `sessionMetrics` field
> There are TWO implementations of segmentedFatigueScore in the codebase (Cloud Function at `functions/src/analytics-utils.ts:252` and client-side at `lib/utils.ts:249`). Both produce scores where **higher = LESS fatigued** (the ratio end/peak is closer to 1). Every coach-facing label says the OPPOSITE.
>
> **Current state after Option B shipped (2026-04-12):**
> - ✅ Cloud Function (`functions/src/index.ts`) writes `segmentedFatigueScore0to10` (10 - rawScore) to `sessionMetrics/{sessionId}` on every analytics run.
> - ✅ Backfill completed 2026-04-12: `scripts/backfill-sessionMetrics.mjs` populated all 1,425 `sessionMetrics` docs with full 80+ field schema. Focused fatigue backfill (`scripts/backfill-fatigue-score.mjs`) also verified — 108 sessions with fatigue data covered.
> - ✅ Agent Mode tools read `sessionMetrics/{sessionId}.segmentedFatigueScore0to10` via `getCanonicalFatigueScore()`, falling back to `flipSegmentedFatigue()` for unbackfilled sessions. Four individual-session sites covered: execQueryGameMetrics, execQuerySessionMetrics, execGetSessionTimeseries, fatigue_comparison chart. Team overview (SUMMARY_STAT_MAP) still uses sync flip.
> - ✅ `metric-docs.ts` describes the canonical field as the source of truth.
> - ⚠️ Inside the `analyze_with_code` sandbox, `<alias>.precomputed.segmentedFatigue.score` is still the RAW value. Code that reads it must apply `10 - score` manually.
> - ❌ Non-Agent-Mode website UI (sessions calendar, dashboard, game analytics page) still reads the raw score from `sensorData`. Migrating those is Step 4 of the full plan and needs a feature flag on `sessions-calendar.tsx`.
> - ❌ Neither formula has been rewritten; both still output the raw direction. The canonical flip happens at write-time in the Cloud Function and backfill script.
>
> **Remaining work:** Website UI migration (Step 4) and final cleanup (Step 5: remove `flipSegmentedFatigue`, remove sandbox warnings). Tracked in [[Agent Memory System#TODO: Option B — canonical segmentedFatigue migration]].
>
> See [[Firestore Collection Audit 2026-04-11#Label/formula bug discovered in segmentedFatigueScore]] for the full original investigation.

---

## 🗃️ Repository quirks — leave them alone

- **Double-nested `src/src/`** in the RN app is intentional (history). Do **not** restructure.
- **Existing wrappers exist for reasons** — extend them, don't bypass them.
- **Don't refactor surrounding code** when fixing something. Bug fixes stay scoped to the bug.

---

## 🧠 Agent Memory System (Cora)

> [!danger] Cora's memory system has its own load-bearing rules
> See [[Agent Memory System]] for the full architecture. The rules below are the ones that cause silent data loss or broken recall if violated.

- **OpenRouter for LLM, Together AI for embeddings.** The provider split is intentional. Don't mix.
- **Always go through `getMemory()`** from `lib/agent/memory/index.ts`. Never call `embed.ts` or `FirestoreAgentMemory` directly from outside the module.
- **No `where()` filters before `findNearest()`** in `store.ts`. That forces a composite vector index that can't be created via the Firebase Console or Firebase CLI without gcloud. Filter in JS post-query and overfetch.
- **The `passage: ` / `query: ` prefix split is mandatory** for the e5 embedding model. `embedPassage()` at write time, `embedQuery()` at read time. Skipping or swapping breaks recall quality.
- **Firestore rejects `undefined` field values.** Never spread an object with optional fields directly into a `set()` / `add()` call. Build payloads field-by-field.
- **`export const maxDuration = 300` at the top of `app/api/agent/route.ts` is load-bearing.** The tool-use loop regularly runs 2–4 minutes; the Vercel default (10s Hobby / 60s Pro) truncates them. Removing this re-introduces the "fresh instance" bug.
- **Session checkpoints are server-only.** `agentSessions/{sessionId}` writes go through `firebase-admin`, not the client SDK. Client just generates a stable `sessionId` and passes it in the POST body.
- **`sanitizeForCheckpoint()` must be called** before writing any `messages[]` array to `agentSessions`. Vision-mode tool results contain base64 blobs that blow through Firestore's 1 MB per-doc cap.
- **Rotate the embedding model at your peril.** Switching `EMBEDDING_MODEL` in `config.ts` invalidates every existing episode embedding. All prior recall breaks until the corpus is re-embedded.
- **Recall context must NOT list prior tool names.** Listing the tools used in past episodes teaches the model to mimic the sequence without re-deriving arguments — seen in production on 2026-04-11 when qwen3.6-plus copied a tool sequence and passed a player NAME as `player_id` because the recall summary hid the preceding ID-lookup step. Keep the recall context to `{coach, date, userQuery, short conclusion}` only. Full story in [[Agent Memory System#5. Recall context must NOT list prior tool names — it teaches mimicry without re-derivation]].
- **Watch the `<tool_call>` detector in logs.** Route.ts now logs `[Agent] Text-mode tool call detected` whenever qwen falls out of native OpenAI function calling and emits `<tool_call>` or `<function=` XML as text. If you see this in production, the recall context or system prompt is likely pushing the model past its native-function-call threshold — trim the context, don't add parsers for the text format.
- **Tool results must NEVER flow through the chat context.** Heavy-data tools (`get_session_timeseries` and any future equivalents) must return **only a schema** — top-line scalars + a `sandboxCatalog` describing what `analyze_with_code` will find in the sandbox. The raw data lives in the sandbox, not in `messages[]`. **Do not add escape-hatch params** like `downsample_to` or `fields` that let the model pull raw data into context anyway — if the model needs the data, the sandbox needs it. Full explanation + the rules for converting new tools in [[Agent Memory System#6. Tool results must NEVER flow through the chat context — schema-first architecture]].
- **Empty iteration = auto-resume, not error.** When the model returns `content === ""` AND `tool_calls === []`, the server checkpoints the in-flight `messages[]` to `agentSessions/{sessionId}` and emits a `resume_needed` SSE event. The client auto-resubmits (up to 3 times) and the server loads the checkpoint to pick up where it left off. Do not change this to a user-facing error without replacing it with something equivalent — the point is that stalls look like a long pause, not a failure.
- **Server emits `heartbeat` SSE events every 10 s** during silent model calls to keep the stream alive through intermediate proxies. Client silently ignores them. Don't remove heartbeats and don't start relying on them in the UI.
- **90-second read timeout on OpenRouter reads** via `AbortController` in `streamCallOpenRouter`. Generous by design — qwen can legitimately take 60+ seconds to serialize an `analyze_with_code` JS payload as tool-call arguments. Don't tighten this without checking what the real p99 serialization time looks like.
- **`max_tokens: 12288`, `reasoning.max_tokens: 16384`** in route.ts are both tuned up from the initial defaults because multi-session analytical questions need more reasoning budget than complex chat does. Lowering these re-introduces empty-iteration stalls.

---

## ✅ Safe-change checklist

Before merging anything in firmware, BLE-adjacent code, Firestore wrappers, or calendar view:

- [ ] Read the existing code
- [ ] Identify all callers / cross-platform mirrors
- [ ] Explain why the change is safe in the PR description
- [ ] Test on real hardware where the chip is involved
- [ ] Verify all three StatsEngine implementations still agree if you touched a metric
