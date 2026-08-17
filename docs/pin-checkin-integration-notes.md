# PIN-triggered auto check-in — exploration notes

**Status: implemented (2026-08-17), pending production verification.** This
shipped as of this session using CourtReserve's official `POST /api/v1/checkins`
API (not the kiosk-session approach originally explored below) — see
[docs/auto-checkin.md](auto-checkin.md) for the end-user-facing setup guide and
`docs/courtreserve-openapi-organization.json` for the API schema. This file is
kept as the research/exploration history — read `auto-checkin.md` first for
current behavior; treat everything below as background on how the design was
reached, not the shipped architecture (the kiosk-session/virtual-kiosk approach
described in this file was superseded once the real Check-In API was found).

## The idea

Today, a member has to do two separate things to get on court: enter their PIN
at the door (CourtPin/UniFi Access), then separately check in at the
`courtreserve-kiosk` touchscreen (scan a QR/barcode). The goal is to collapse
those into one action — entering the door PIN also triggers the CourtReserve
check-in automatically, no kiosk touch needed.

## Architecture sketch

```
Member enters PIN at door
        |
        v
UniFi Access console fires "Door unlocked" event
        |
        v
Alarm Manager -> Webhook -> [new listener process]
        |
        v
Listener resolves PIN/actor -> CourtReserve OrganizationMemberId
        |
        v
Listener calls CourtReserve's SubmitCheckIn (same call courtcheckin/src/courtreserve.js
already makes) with that member's barcode
```

The two existing projects each already have half of what's needed:
- **CourtPin** (`index.js`) already maps `OrganizationMemberId` <-> PIN, at least
  when `PIN_MODE=static` (the PIN *is* the member ID, or a truncated form of it —
  see `STATIC_PIN_LENGTH` in CLAUDE.md). It also already holds a UniFi bearer
  token and talks to the same UniFi Access console.
- **courtreserve-kiosk**'s `courtcheckin` app already knows how to submit a
  check-in: `submitCheckIn(barcode, method)` in
  `courtreserve-kiosk/courtcheckin/src/courtreserve.js:139`, which POSTs to
  `https://app.courtreserve.com/Kiosk/Home/SubmitCheckIn`. It authenticates via
  a kiosk session cookie (`CR_KIOSK_ID`/`CR_PIN_CODE` in that project's `.env`),
  not the CourtPin API key.

The missing piece is the listener in the middle. It doesn't need to live on
either existing device necessarily — it just needs to (a) be reachable by the
UniFi Access console over the LAN, and (b) be able to call CourtReserve's
check-in endpoint (needs outbound internet, which any on-site device has).

## Confirmed via research (this session)

- UniFi Access supports a real webhook mechanism: **Alarm Manager** → create an
  alarm → Trigger `Door unlocked` → Scope the door/reader → Action `Webhook` →
  Delivery URL (accepts plain `http://`) → Delivery Method `POST`.
- A working third-party integration
  ([fvhemert/Home-Assistant---Unifi-Access---Webhook-integration](https://github.com/fvhemert/Home-Assistant---Unifi-Access---Webhook-integration))
  confirms the JSON payload includes:
  - `id`: e.g. `"access.unlocks.location_unlocked"` — event type
  - `unlock_method` / `unlock_method_text`: e.g. `"touch_pass"` / `"Touch Pass"`
    — this is presumably where a PIN entry would show something like
    `"pin_code"`, but **not yet confirmed** — we only triggered a remote/app
    unlock, not an actual PIN entry, this session.
  - `user` / `trigger_user`: UUID identifying the person — **this is a UniFi
    Access user/visitor UUID, not the CourtReserve `OrganizationMemberId`
    directly.** Correlating it back to a member requires either matching it
    against the UniFi visitor ID CourtPin stored when it called `createVisitor()`,
    or (simpler) just trusting the PIN plaintext if we can get it — but per
    CLAUDE.md, "UniFi's API only returns a hash on future reads" for PINs, so
    the webhook payload's `user` UUID is likely the only real correlation key,
    not the PIN digits themselves.
  - `location`: UUID of the door/device.
  - `direction`: e.g. `"entered"`.
- Separately, the official UniFi Access developer API type docs
  ([hjdhjd/unifi-access](https://github.com/hjdhjd/unifi-access/blob/main/docs/access-types.md))
  confirm a **websocket** channel also exists
  (`wss://<host>/api/v1/developer/devices/notifications`), pushing
  `AccessEventPacket` objects with an `event` string — confirmed example:
  `access.data.device.remote_unlock`. Not established whether keypad PIN entries
  use this same channel/event name or a different one. Webhook is the better-
  documented path for this use case; websocket is a fallback if webhooks turn
  out to be insufficient.

## Confirmed on-site 2026-08-17: real PIN-unlock payload captured

A follow-up on-site session (laptop joined the club's Wi-Fi directly, so no
reachability issue this time) got a real PIN-code door unlock captured by the
throwaway listener. Full payload:

```json
{
  "alarm_id": "01a010b9-97d2-7373-afd1-39836e3f8ea3",
  "events": [{
    "id": "access.unlocks.location_unlocked",
    "scope": { "locations": "4612e928-5a28-4427-9177-f519f80327e8" },
    "device": "1c0b8bcef5ae",
    "device_name": "Main Entry - Entry",
    "location": "4612e928-5a28-4427-9177-f519f80327e8",
    "location_name": "Main Entry",
    "user": "eafce5a9-c882-4244-b817-4f34f32d9f0f",
    "user_name": "Joel Peluchette",
    "user_type": "visitor",
    "admin": "",
    "time": "1786992505",
    "emergency_mode": "",
    "direction": "entered",
    "unlock_method_text": "PIN",
    "credential_type": "PIN_CODE",
    "credential_value": "ddc59896"
  }],
  "data": { "custom_content": "" }
}
```

This resolves the two previously-open questions:

- **`unlock_method_text` does distinguish PIN entry** — confirmed value is
  literally `"PIN"` (paired with `credential_type: "PIN_CODE"`), distinct
  from the `"touch_pass"` seen on a remote/app unlock. A listener can filter
  on this reliably.
- **`user_name` comes through as a human-readable name** (`"Joel Peluchette"`)
  in addition to the `user` UUID — may be usable as a secondary/fallback
  correlation signal, though the UUID is still the more reliable key.
- `credential_value: "ddc59896"` is present but is 8 hex characters, not a
  plaintext numeric PIN — almost certainly a credential reference/hash, not
  the raw PIN digits. Consistent with CLAUDE.md's note that UniFi never
  returns plaintext PIN values on reads. **Don't rely on this field to
  recover the PIN itself.**

**Confirmed via Railway logs (2026-08-17, same session):** CourtPin's
production log (Railway — confirmed still the live host, the Pi migration
described in CLAUDE.md has not actually happened yet) shows:

```
[2026-08-17T18:46:03.096Z] [INFO] ✅ Player processed successfully
{"reservationId":"59126283","memberId":1788252,
 "visitorId":"eafce5a9-c882-4244-b817-4f34f32d9f0f",
 "email":"peluchettejoel@gmail.com","pin":"8252","court":"Court #1",
 "reservationStart":"8/17/2026, 3:00:00 PM"}
```

`visitorId` here is an **exact match** for the webhook's `user` field. Full
correlation chain is now confirmed end-to-end: webhook `user` UUID →
CourtPin's stored `visitorId` (in `state.processed[reservationId:memberId]`)
→ `memberId` → CourtReserve `OrganizationMemberId`. A listener can reliably
look up `state.json` (or a similar in-memory index) by `visitorId` to recover
the member ID once it receives a PIN-unlock webhook.

## Earlier attempt (off-site, unsuccessful) — for reference

An earlier attempt failed because the person testing was not on-site, so the
listener (running on their Windows machine) was on a completely different LAN
than the UniFi console — UniFi Access can't route a webhook to a private
`192.168.x.x`/on-site LAN address on a network it has no path to, regardless
of whether the *trigger* (a remote/app unlock) was fired from off-site or
on-site. The trigger location doesn't matter; the **listener's** network
reachability from the console is what matters. This is why the follow-up
session ran the listener from a laptop joined directly to the club's Wi-Fi.

## CourtReserve-side identity — confirmed 2026-08-17

`submitCheckIn(barcode, method)` was called directly on the kiosk Pi
(`courtcheckin` module, real production session) with the plain
`OrganizationMemberId` (`'1788252'`) as `barcode`:

```
node test-checkin.js
...
Result: { "isValid": true, ..., "data": "...<h1 class=\"member-fullname\">Joel Peluchette</h1>...
  Court Reservation 3:00 PM - 4:00 PM Court #1..." }
```

**Confirmed: CourtReserve's `SubmitCheckIn` endpoint accepts a plain
`OrganizationMemberId` directly as `barcode` — no separate CourtReserve-issued
barcode value is needed.** The response correctly identified the member by
name and matched their actual active reservation (time + court). This means
a listener can call `submitCheckIn(memberId)` using the same `memberId`
CourtPin already has in `state.processed`, with no additional lookup step.

## Deployment-target uncertainty — resolved

**Confirmed 2026-08-17: CourtPin is still running on Railway (cloud)**,
despite CLAUDE.md's "Primary deployment target" section describing the Pi
migration as already done. Verified directly via the Railway dashboard
(`courtpin.up.railway.app`, service showing "Online", production logs
actively flowing). The Pi migration described in CLAUDE.md has not actually
happened for this install as of this date — don't trust that doc section at
face value without reconfirming. This matters for the eventual listener
design: it cannot assume LAN-local reachability to the UniFi console the way
an on-site Pi would have; a Railway-hosted listener needs the same public
webhook delivery path CourtPin itself would need (which is exactly the
port-forwarding problem CLAUDE.md describes hitting previously).

## Plan / progress

1. ~~Confirm which device(s) are actually on the club's LAN right now.~~
   Done 2026-08-17 — tested with a laptop joined directly to the club's Wi-Fi
   rather than an always-on device; worked fine as a one-off test, but see
   "Open item" below on whether the real listener needs an always-on device.
2. ~~Run the listener on-site, point the Alarm Manager Delivery URL at it.~~
   Done 2026-08-17.
3. ~~Enter a real PIN at the door and capture the actual payload.~~ Done
   2026-08-17 — see confirmed payload above. `unlock_method_text: "PIN"` does
   distinguish PIN entry from other methods.
   - ~~Does the `user` UUID match the UniFi visitor ID CourtPin stored from
     `createVisitor()`?~~ **Confirmed 2026-08-17 via Railway logs** — exact
     match, see confirmed correlation chain above.
4. ~~Separately test `submitCheckIn()` with a known member ID as `barcode`.~~
   **Confirmed 2026-08-17** — see "CourtReserve-side identity" above. Plain
   `OrganizationMemberId` works with no translation needed.

Both prerequisites are now confirmed. **Both core unknowns for this
integration are resolved:**
- Webhook payload reliably distinguishes a PIN-code unlock
  (`unlock_method_text: "PIN"`) and its `user` UUID matches CourtPin's stored
  `visitorId` exactly, giving a clean path to `memberId`.
- `submitCheckIn(memberId)` works directly with the member ID CourtPin
  already has on hand — no extra CourtReserve barcode lookup needed.

5. **Next (design, not implementation):** design the actual listener process
   — where it lives (Railway is still the live host, not an on-site Pi, which
   affects webhook reachability — see "Deployment-target uncertainty" above),
   whether it's a new small service or folded into CourtPin, error handling
   for a member who's not actually registered for anything today, etc. Per
   this project's house rules, **no implementation without an explicit
   go-ahead** after the design is agreed on.
