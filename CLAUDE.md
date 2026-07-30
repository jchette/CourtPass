# CLAUDE.md

This file gives Claude Code the context needed to work in this repository without re-deriving it from scratch. Read this fully before making changes.

---

## What this project is

CourtPin connects [CourtReserve](https://courtreserve.com) (court/event booking platform) with [UniFi Access](https://ui.com/door-access) (door control) to automatically grant members building access tied to their reservations and event registrations — no staff involvement required.

It is a single-file Node.js application (`index.js`) intentionally kept dependency-light and framework-free. There is no build step, no TypeScript, no database. State is a flat JSON file.

**Primary deployment target right now:** a Raspberry Pi running on the same local network as the UniFi console (no port forwarding needed — see "Deployment context" below). Railway (cloud) is the secondary/alternate deployment path, documented as an option for setups that don't want local hardware.

---

## Architecture — one file, clearly sectioned

Everything lives in `index.js`, organized into these sections in order:

1. **Configuration** — `config` object built entirely from `process.env`. No hardcoded values anywhere.
2. **Helpers** — `loadState`, `saveState`, `log`, `toEpoch`, `fmtDate`, `fmtLocalDatetime`, `deriveStaticPin`
3. **HTTP Clients** — `courtreserve` (axios, Basic Auth) and `unifi` (axios, Bearer token, self-signed cert bypass)
4. **Email** — `sendUnlockNotificationEmail`, `sendUnlockNotificationSms`, `sendAccessEmail`, `_sendEmail` (shared transport helper), `sendAccessSms`
5. **CourtReserve API** — `fetchTodaysReservations`, `fetchTodaysEventRegistrations`
6. **UniFi Access API** — `generatePin`, `createVisitor`, `assignPin`, `deleteVisitor`, `unlockDoor`, `fetchExpiredUnifiVisitors`
7. **Core Processing** — `processReservation`, `processEvents`, `cleanupExpiredVisitors`, `runCycle`
8. **Admin Server** — `startAdminServer` and session helpers (`generateSessionId`, `isAuthenticated`, `getSessionId`)
9. **Admin HTML** — `loginPage`, `dashboardPage` (return raw HTML strings, no templating engine)
10. **Entrypoint** — `validateConfig`, `main`

When adding a feature, find the matching section and keep new code there rather than appending to the bottom of the file.

---

## Core flow (read this before touching processing logic)

Every minute, `runCycle()`:

1. Fetches today's active **reservations** from CourtReserve → `processReservation()` per reservation, per player
2. Fetches today's active **event registrations** from CourtReserve → `processEvents()`, grouped by event session
3. Runs `cleanupExpiredVisitors()` — two passes: (a) delete visitors tracked in `state.json` past their end time, (b) query UniFi directly for any expired visitor not in state (catches orphans from state resets)

### Reservations
One UniFi Visitor + one PIN per player on the reservation. State key: `reservationId:memberId`.

### Events — three access modes, controlled by `EVENT_ACCESS_MODE`
- **`pin_individual`** (default) — same as reservations, one Visitor + PIN per registrant. State key: `evt:eventId:eventDateId:memberId`
- **`pin_shared`** — one Visitor + PIN for the whole event, same PIN sent to all registrants. State key: `evt:eventId:eventDateId:shared`
- **`unlock`** — unlocks door(s) in `UNIFI_RESOURCES` for the event duration, no PINs. State key: `evt:eventId:eventDateId:unlock`

**Important pattern used in both `pin_shared` and `unlock`:** visitor/door-unlock creation and per-registrant notification are deliberately split into two separate steps. Step 1 (create visitor / unlock door) only runs once per event and is gated by `!state.processed[key]`. Step 2 (notify registrants) runs **every cycle** and tracks who's been notified via a `notified: []` array stored on the same state entry, checking each registrant's `OrganizationMemberId` against it. This was a real bug fixed in production — a first version gated the whole block (including notification) behind the "already processed" check, which meant new registrants who joined after the first cycle never got notified. **Do not re-merge these steps if extending this logic.**

---

## PIN generation modes

Controlled by `PIN_MODE`:
- `random` (default) — calls UniFi's PIN generation endpoint, works regardless of PIN length constraints
- `static` — uses the CourtReserve `OrganizationMemberId` as the PIN via `deriveStaticPin()`. Falls back to random if `memberId` is falsy (guests booking without a member ID, e.g.) — this fallback exists in both `processReservation` and `processEvents` and logs a warning only in the reservation path (the event path fails silently by design since guest bookings on events are common and not worth logging every time).

`STATIC_PIN_LENGTH` controls truncation when `PIN_MODE=static`:
- `full` (default) — uses the entire member ID regardless of length
- a number (e.g. `4`) — truncates to the last N digits via `.slice(-N)`

**Known real-world issue:** UniFi Access sites enrolled in a **UniFi Fabric** (multi-site management) have a confirmed Ubiquiti bug where PIN Length is locked to Fixed 4-digit and cannot be changed to Variable Length, even though the UI shows the option. This is why `STATIC_PIN_LENGTH` exists — set it to `4` to work around the bug. Confirmed with UniFi support as a platform bug, not fixable from CourtPin's side.

---

## Email transport — priority logic

`_sendEmail()` picks the transport with a simple, non-configurable priority: **if `RESEND_API_KEY` has any truthy value, Resend is used — full stop, regardless of whether SMTP variables are also populated.** SMTP (via nodemailer) is only used when `RESEND_API_KEY` is empty/unset. There is no merging or fallback between the two. If a user reports "I filled in SMTP but it's still using Resend," the fix is telling them to blank out `RESEND_API_KEY`, not a code change.

Resend is recommended for cloud hosting (Railway blocks outbound SMTP ports 25/465/587 on free/hobby tiers). SMTP is recommended for local hosting (Pi, NAS) where there are no port restrictions and clubs likely already have email hosting through their domain provider.

---

## Deployment context

**Local (Pi) is now the primary documented path for this specific installation.** The club hit real, unresolvable port-forwarding issues (double-NAT/double-firewall) trying to let Railway (cloud) reach the on-prem UniFi console, and migrated to a Raspberry Pi on the same LAN as UniFi. This eliminates port forwarding entirely for the CourtReserve↔UniFi communication — `UNIFI_HOST` is set to the local IP (e.g. `https://192.168.1.1:12445`) and outbound-only internet access (CourtReserve API, email, SMS) is all that's needed.

Process management on the Pi is via **PM2** (`pm2 start index.js --name courtpin`, `pm2 save`, `pm2 startup`). Updates are `git pull && pm2 restart courtpin` — there is no CI/CD on this path, unlike Railway which auto-deploys on push.

Railway remains documented as the recommended path for **other installs** that don't want on-site hardware or SSH comfort — see `docs/hosting.md` for the full comparison across all six supported hosting methods (Railway, Raspberry Pi, Docker, Synology/QNAP NAS, VPS, Windows PC).

A `Dockerfile` is intentionally **not** committed to the repo root — Railway auto-detects Node.js via Nixpacks, and a committed Dockerfile overrides that detection and causes build failures (`npm ci` requiring a lockfile that doesn't exist, since this repo doesn't commit `package-lock.json`). The Dockerfile content needed for Docker-based self-hosting is documented inline in `docs/hosting.md` for users to create themselves.

---

## CourtReserve API notes

- Auth is HTTP Basic: `CR_ORG_ID` as username, `CR_API_KEY` as password, on every request via the `courtreserve` axios instance.
- `GET /api/v1/reservationreport/listactive` — requires `ReservationReport` role with Read permission. Returns `Players[]` per reservation with `OrganizationMemberId`, `FirstName`, `LastName`, `Email`, `Phone`.
- `GET /api/v1/eventregistrationreport/listactive` — requires `EventRegistrationReport` role with Read permission (separate role from reservations, must be enabled explicitly on the API key). Returns one row per registrant with `EventId`, `EventDateId`, `EventName`, `StartTime`, `EndTime`, `OrganizationMemberId`, `FirstName`, `LastName`, `Email`, `Phone`.
- Both endpoints are queried for **today's date range only** (`startOfDay` to `endOfDay` in local time), not a rolling window. This is intentional — `StartTime`/`EndTime` come back with no timezone suffix, so the query and all time math assumes the `TZ` env var matches the facility's actual timezone. Getting `TZ` wrong is the most common real-world bug reported (see `docs/troubleshooting.md`).
- Date range parameters cannot exceed 31 days per CourtReserve's API constraints (documented behavior, not a CourtPin limitation).

## UniFi Access API notes

- Auth is Bearer token via the `unifi` axios instance, with `rejectUnauthorized: false` in the httpsAgent — UniFi's console uses a self-signed cert by default, this is expected and documented by Ubiquiti, not a security oversight to "fix."
- Required token scopes: `view:credential` (PIN generation), `edit:visitor` (create/update/delete visitors).
- `unlockDoor()` handles both `door` and `door_group` resource types from `UNIFI_RESOURCES` — for a `door_group` it first fetches the group's topology to enumerate individual door IDs, then unlocks each one separately (there's no single "unlock this whole group" endpoint).
- Once a PIN is assigned, UniFi's API only returns a hash on future reads — the plaintext PIN is unrecoverable from UniFi's side. This is why CourtPin persists the PIN in `state.json` and logs at creation time — it's the only source of truth for the admin portal's PIN lookup/resend feature.

---

## State file (`state.json`)

Flat JSON, single top-level key `processed`, keyed by the state keys described above. No database, no migrations. On Railway this lives at `/tmp/state.json` (ephemeral — cleared on container restart, which is why the UniFi-side orphan cleanup pass in `cleanupExpiredVisitors()` exists as a safety net). On the Pi it can live anywhere persistent, e.g. `~/courtpin/state.json`.

If you need to force CourtPin to reprocess something (e.g. for testing), the state file can be edited directly or the `STATE_FILE` path changed to point somewhere fresh — there's no CLI or admin-portal action to do this currently.

---

## Admin portal

Plain Node `http` server (no Express), started by `startAdminServer()`. Session auth is a random hex token stored in an in-memory `Set` (`sessions`) — sessions are lost on process restart by design (acceptable for a low-traffic internal admin tool). Cookie name: `courtpin_session`.

Routes:
- `GET /admin/login`, `POST /admin/login` — password check against `ADMIN_SECRET`
- `GET /admin/logout`
- `GET /admin` — dashboard, requires auth
- `POST /admin/resend` — re-sends PIN email/SMS for a given `reservationId`/`playerId` pair, requires auth
- `GET /health` — no auth, basic liveness check

Dashboard cards show a yellow badge (`Event`, `Event (shared PIN)`, or `Event (door unlock)`) to visually distinguish event-derived access from reservation-derived access — see the `isEvent`/`typeLabel` logic in `dashboardPage()`.

---

## Things NOT to do without asking first

- **Don't rewrite large chunks of `index.js` speculatively.** This project has an explicit house rule (set by the project owner mid-session) that code changes only happen on explicit request — questions about "is X possible" are not requests to implement X. When in doubt, ask before editing.
- **Don't re-add a root-level `Dockerfile`** without confirming Railway deployment isn't the active target — it breaks Nixpacks auto-detection there (see "Deployment context" above).
- **Don't merge the two-step pattern** in `pin_shared`/`unlock` event processing back into a single gated block — this reintroduces the late-registrant notification bug described above.
- **Don't add STOP/opt-out language to SMS.** It was deliberately removed — see git history around the SMS template rewrite — because these are transactional access messages, not marketing, and STOP replies would unsubscribe someone from future PIN delivery in a way that causes real confusion at the door.
- **Don't assume `.env` exists in the repo.** It's gitignored. `env.example` is the template; real credentials live only in Railway's Variables tab or a local `.env` on the Pi.

---

## Versioning and release notes

Releases are tagged on GitHub (`github.com/jchette/courtpin/releases`) following semver:
- Patch (`v1.0.x`) — bug fixes
- Minor (`v1.x.0`) — new backward-compatible features (e.g. `v1.1.0` added `PIN_MODE`)
- Major (`vx.0.0`) — significant feature additions (e.g. `v2.0.0` added event support)

**House rule:** only tag a release after the change has been confirmed working in production by the project owner — not immediately after writing the code. If uncertain whether something's been tested, ask before proposing a version bump. Release notes use GitHub-flavored markdown with a consistent structure: `## What's new/changed`, code blocks for env vars, `## Files changed`, `## How to update`, `## Verified`.

---

## Documentation map

- `README.md` — lean entry point, links out to `docs/`
- `docs/configuration.md` — every env var, plus a copy-paste Railway raw-editor block
- `docs/hosting.md` — all six hosting methods (Railway, Pi, Docker, NAS, VPS, Windows)
- `docs/unifi-setup.md` — API token creation, door group IDs, port forwarding, Cloudflare Tunnel
- `docs/email-setup.md` — Resend setup, SMTP setup (including provider-specific settings and Gmail app passwords), local SMTP servers
- `docs/admin-portal.md` — using the PIN lookup/resend portal
- `docs/troubleshooting.md` — symptom → cause → fix, including the UniFi Fabric PIN-length bug

Keep these in sync when changing behavior — this project has been burned before by docs drifting out of date after a code change (see the SMS STOP-line and PIN_MODE doc-sync fixes in project history). When you change a env var's behavior or add a new one, update `env.example`, `docs/configuration.md`, and the README feature list in the same pass.
