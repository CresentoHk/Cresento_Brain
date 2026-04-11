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

### `users/{uid}`

- Profile (name, email)
- **`role`** — `"player"` or `"coach"`. Drives navigation in all clients.
- Session counters
- Privacy settings (configured by coaches on the website, read by mobile)

### `sensorData/{docId}`

> [!warning] Paged — read paging rules below
> Each session is stored across MULTIPLE `sensorData` documents, not one. The page schema is shared and load-bearing.

- One physical recording session = N pages
- Each page ≤ **700 KB** (Firestore hard limit is 1 MB; the cap reserves headroom)
- Pages are atomic batched on write so partial sessions don't appear
- Decoded by RN, iOS, AND the website — keep the schema stable

### `teams/{teamId}`

- Coach team info — players, settings, branding

### `players/{playerId}`

- Player profiles (separate from `users` because a player isn't always a registered user)

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
