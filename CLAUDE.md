# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-room private chat for **two people**, self-hosted on a Mac. Text, photo,
video, audio and file messages over a secret link, protected by an access code.
Comments and user-facing strings are in Portuguese (pt-BR); keep new ones in
Portuguese too.

## Commands

- `npm start` — run the server (`node server.js`). Listens on `127.0.0.1:$PORT`
  (default 4177). Requires `ROOM_SLUG` (≥16 chars) in `.env` or it exits.
- `npm install --omit=dev` — install runtime deps (express, multer, archiver).
- `scripts/install.sh` — generates `.env` with a random `ROOM_SLUG`, then
  installs a `launchd` agent (`com.privatechat.server`) that keeps the server
  running and restarts it on boot. Used for real deployment on the Mac.
- `scripts/setup-ngrok.sh <authtoken> <domain>` — adds a second `launchd` agent
  exposing the local port through a fixed ngrok domain.
- `scripts/uninstall.sh` — unloads both `launchd` agents. Does **not** touch
  `data/`.

There is **no test suite, linter, or build step** despite some code comments
referencing "tests" (e.g. the `EPHEMERAL_TTL_MS` env override exists for tests
that were never committed). To iterate locally, run `npm start` with a hand-made
`.env` and hit `http://127.0.0.1:4177/c/<slug>`.

Useful env vars: `PORT`, `ROOM_SLUG`, `DATA_DIR`, `MAX_UPLOAD_MB` (default 300),
`EPHEMERAL_TTL_MS` (default 10000), `COOKIE_INSECURE=1` (drop `Secure` flag so
cookies work over plain-HTTP localhost).

## Architecture

### Encryption model (the core constraint)

The access code is **never stored anywhere** — not plaintext, not hashed. It only
exists in memory during the moment someone logs in.

- On first login ever, `Store.unlock(code)` bootstraps an empty conversation
  encrypted under whatever code the first visitor typed. That visitor defines the
  access code.
- `lib/crypto.js`: `deriveKey` runs scrypt(code, salt) → 32-byte key. `salt.bin`
  is public, stored in `data/`. `encryptBuffer`/`decryptBuffer` are AES-256-GCM
  with layout `[iv(12)][authTag(16)][ciphertext]`. A GCM auth failure on decrypt
  **is** how a wrong code is detected (`WrongCodeError`).
- Everything on disk is ciphertext: `data/store.enc` (the whole message log as
  one JSON blob) and `data/media/<id>.enc` (one file per upload). Writes are
  atomic (temp file + rename), mode `0600`.
- `lib/auth.js`: on successful login the derived key is held in an in-memory
  `sessions` Map keyed by a random session id; the client gets an HMAC-signed
  HttpOnly cookie (`pc_session`, 30-day TTL, `Path` scoped to the room).
  `SESSION_SECRET` is regenerated every process start, so **restarting the server
  invalidates every session** — by design, since the key lives only in RAM.
- Authenticated requests never re-derive the key. `loadWithSessionKey(req)` in
  `server.js` reads `store.enc` fresh off disk and decrypts it with
  `req.session.key` on every request (no long-lived in-memory copy of the
  conversation — avoids one request clobbering the other person's concurrent
  write).

### Routing / obscurity

- Everything lives under `/c/<ROOM_SLUG>`. A top-level middleware in `server.js`
  returns bare `404` for any path that isn't exactly the room path or a child of
  it — a wrong/absent slug gets no hint that a chat exists here.
- `roomRouter` serves `public/` statically plus the `/api/*` endpoints.
- `lib/env.js` is a tiny dependency-free `.env` loader (`KEY=VALUE` only).

### Real-time

- `GET /api/stream` is a Server-Sent Events endpoint. A single Node
  `EventEmitter` (`bus`) fans out events: `message`, `message-updated`,
  `message-deleted`, `message-viewed`, `cleared`, `presence` / `presence-init`.
- **Presence** (`onlineConnections` Map) and **ephemeral expiry timers**
  (`pendingExpiry` Map) are in-memory only, same "wiped on restart" philosophy as
  sessions.
- Frontend (`public/app.js`) holds one `EventSource` with silent auto-reconnect.
  All fetches go through `api(path)` which prefixes the room path derived from
  `location.pathname`.

### "Visualização única" (view-once media)

The most intricate feature — spread across `sanitizeMessage`, the `/view`
endpoint, `scheduleExpiry`, and `reconcileEphemeral` in `server.js`:

- Ephemeral image/video messages have their `mediaId` (and `filename`,
  `mimeType`, `size`, `width`, `height`) **stripped** by `sanitizeMessage` from
  every list response and every SSE broadcast.
- `POST /api/messages/:id/view` is the single deliberate reveal point: it returns
  the real `mediaId` once, arms a `setTimeout` expiry (`EPHEMERAL_TTL_MS` after
  first open), and persists `viewedAt`. The original sender is forbidden from
  opening it.
- On expiry the message is hard-deleted: media file unlinked, text/media fields
  removed, `expiredEphemeral: true` set.
- `reconcileEphemeral` runs on every authenticated load to self-heal timers lost
  to a restart — it catches up any window that finished while the process wasn't
  running, and re-arms live timers for windows still open.
- Ephemeral media is **never** included in the export zip, in any state.

### Other message behaviors

- **Replies**: `buildReplySnapshot` freezes sender + a short snippet at reply
  time; the quote keeps showing that even if the original is edited/deleted.
- **Delete**: either person can delete any message (2-person trusted room);
  `deletedBy` is recorded and shown on the placeholder.
- **Link previews**: `scheduleLinkPreview` runs *after* the message is saved and
  broadcast (fire-and-forget), fetches only the `<head>` of the first URL,
  extracts OpenGraph tags, and patches the bubble via `message-updated`. Any
  failure just means no card. `isBlockedPreviewHost` blocks obvious
  internal/loopback hosts — it is a courtesy filter, **not** a full SSRF defense
  (no DNS-rebinding protection); acceptable only because both users are already
  fully trusted. `stripTrailingPunctuationServer` must stay in sync with
  `stripTrailingPunctuation` in `app.js`.
- **Media dimensions**: client reads intrinsic width/height before upload and
  sends them so the bubble reserves layout space and doesn't jump on load.

### Decoy / trap password

`DECOY_CODE` (hardcoded in `server.js`) makes `/api/login` return
`{ ok: true, decoy: true }` — no session, no rate-limit hit, no failure log, no
contact with the store. The frontend then shows a fake "tool under development"
screen (`#decoy-screen`). Purpose: someone who shouldn't be poking around but
knows/guesses this specific code sees nothing that reveals a real chat exists.

### Rate limiting

`lib/auth.js`: 6 wrong codes from an IP → 10-minute lock (`isLocked` /
`registerFailure`). Cleared on success. In-memory `attempts` Map.

## Export

`GET /api/export` streams a zip (`archiver`) with `conversa.json`, `conversa.txt`
and `midias/`. The export is **decrypted** — that's the point (portable backup) —
so it's outside the security model once written.

## Frontend

`public/` is plain HTML/CSS/JS, no framework, no build. `app.js` is one large
IIFE. Notable client-only bits: love-message counter and Konami-style easter eggs
persisted in `localStorage`; `NAME_KEY` stores the user's display name; heavy use
of the Visual Viewport API to fight the iOS keyboard/URL-bar layout jitter.
