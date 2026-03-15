# CourtPass

![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)
![Node.js](https://img.shields.io/badge/Node.js-18%2B-green.svg)
![Platform: Railway](https://img.shields.io/badge/Deploy-Railway-blueviolet.svg)

**Automated building access for court sports clubs.**

CourtPass connects [CourtReserve](https://courtreserve.com) with [UniFi Access](https://ui.com/door-access) to automatically send members a door PIN before their reservation — no staff involvement required. When a member books a court, CourtPass creates a time-limited visitor in UniFi Access, generates a unique PIN, and delivers it by email and optional SMS. After the reservation ends, access is automatically revoked.

Built for tennis, pickleball, squash, and racquetball clubs running CourtReserve for reservations and UniFi Access for door control.

---

## What it does

Automatically sends building **access PINs** to CourtReserve members before their court reservation via email and optional SMS. Each member is registered as a temporary **Visitor** in UniFi Access with a randomly generated PIN. Admins can look up and resend PINs via a mobile-friendly branded web portal.

---

## Features

- Polls CourtReserve every minute for today's reservations
- Creates time-scoped UniFi Access Visitors automatically
- Generates and assigns a unique PIN per member per reservation
- Sends branded HTML email with PIN and access window details
- Optional SMS via Twilio
- Configurable early-access buffer (members enter X minutes before their slot)
- Auto-cleanup — Visitor records deleted after reservation ends
- Mobile-friendly admin portal for PIN lookup and one-tap resend
- PIN stored in logs and state file for admin reference
- Full club branding via environment variables — no code changes needed

---

## How it works

```
Every minute:
  ├─ Fetch today's active reservations from CourtReserve
  │    └─ GET /api/v1/reservationreport/listactive
  │
  ├─ For each reservation within the notification window (not yet processed):
  │    └─ For each player with an email address:
  │         ├─ Create UniFi Visitor (access window = reservation - ACCESS_BUFFER_MINUTES → end)
  │         ├─ Generate random PIN via UniFi Credential API
  │         ├─ Assign PIN to Visitor
  │         ├─ Send branded PIN email via Resend API
  │         └─ Send PIN SMS via Twilio (optional)
  │
  └─ Cleanup: delete Visitors whose reservation ended > CLEANUP_BUFFER_MINUTES ago
```

Multi-player reservations are fully supported — every player on a booking gets their own Visitor and unique PIN.

---

## Example timeline

With defaults (`NOTIFY_MINUTES_BEFORE=60`, `ACCESS_BUFFER_MINUTES=30`):

```
10:00 AM  Member books a 12:00 PM reservation in CourtReserve

11:00 AM  Integration detects reservation is 60 min away
           → Creates UniFi Visitor (active 11:30 AM – 1:00 PM)
           → Generates PIN e.g. 67203419
           → Sends email: "Your Club Name Access PIN - Sun Mar 15, 12:00 PM"
           → Sends SMS (if enabled)

11:30 AM  PIN becomes active — member can enter the building

12:00 PM  Reservation starts

 1:00 PM  Reservation ends

 1:15 PM  UniFi Visitor deleted, PIN revoked
```

---

## Prerequisites

| Requirement | Notes |
|---|---|
| Node.js 18+ | `node --version` to check |
| CourtReserve API access | Requires **ReservationReport** role with Read permission |
| UniFi Access 1.9.1+ | API token needs `view:credential` + `edit:visitor` scopes |
| Resend account | Free at resend.com — sends email via HTTPS |
| Twilio account | Optional — free trial at twilio.com for SMS |

---

## Quick Start (local)

```bash
npm install
cp env.example .env
# Edit .env with your credentials
npm start
```

---

## Deploying to Railway

Railway is the recommended platform — it keeps the integration running 24/7 with zero server management.

1. Push the project to a **private GitHub repo**
2. Sign up at **railway.app** → New Project → Deploy from GitHub repo
3. Select your repo
4. Go to **Variables** tab → Raw Editor and paste your variables (see below)
5. Go to **Settings → Networking → Generate Domain** to get your public URL
6. Railway auto-deploys on every GitHub push

**Important:** Your UniFi console must be reachable from Railway. Options:
- **Port forwarding** — forward external port `12445` on your router to your UniFi console's local IP on port `12445`
- **Cloudflare Tunnel** — more secure, no open ports required

---

## Railway — Raw Editor (copy & paste)

In Railway → Variables → **Raw Editor**, paste this block and fill in your values:

```
CR_BASE_URL=https://api.courtreserve.com
CR_ORG_ID=your_org_id_here
CR_API_KEY=your_api_key_here
UNIFI_HOST=https://YOUR_PUBLIC_IP:12445
UNIFI_API_TOKEN=your_unifi_api_token_here
UNIFI_RESOURCES=
RESEND_API_KEY=re_xxxxxxxxxxxxxxxxxxxx
EMAIL_FROM=noreply@yourclub.com
TWILIO_ENABLED=false
TWILIO_ACCOUNT_SID=ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
TWILIO_AUTH_TOKEN=your_auth_token_here
TWILIO_FROM_NUMBER=+13045550100
BRAND_CLUB_NAME=Your Club Name
BRAND_TAGLINE=Court Access
BRAND_HEADER_COLOR=1a56db
BRAND_ACCENT_COLOR=1a56db
BRAND_WEBSITE=https://yourclub.com
BRAND_PHONE=555-555-0100
BRAND_ADDRESS=123 Main St, Your City ST
ADMIN_PORT=3000
ADMIN_SECRET=choose_a_strong_password_here
NOTIFY_MINUTES_BEFORE=60
ACCESS_BUFFER_MINUTES=30
CLEANUP_BUFFER_MINUTES=15
TZ=America/New_York
STATE_FILE=/tmp/state.json
PORT=3000
```

**Note:** Enter hex colors **without** the `#` symbol — Railway treats `#` as a comment character. The code adds it automatically.

---

## Variable Reference

| Variable | Required | Description |
|---|---|---|
| `CR_ORG_ID` | ✅ | CourtReserve org ID — visible in admin portal URL |
| `CR_API_KEY` | ✅ | CourtReserve API key — Admin → Settings → API Access |
| `UNIFI_HOST` | ✅ | Full URL with port e.g. `https://YOUR_PUBLIC_IP:12445` |
| `UNIFI_API_TOKEN` | ✅ | UniFi Portal → Access → Settings → General → Advanced → API Token |
| `UNIFI_RESOURCES` | ⚠️ | Doors/groups members can access. Format: `door_group:ID` or `door:ID`. Leave empty for testing. |
| `RESEND_API_KEY` | ✅ | resend.com → API Keys → Create API Key |
| `EMAIL_FROM` | ✅ | Sender address from a verified Resend domain. Use `onboarding@resend.dev` for testing. |
| `TWILIO_ENABLED` | — | Set `true` to activate SMS. Default: `false` |
| `TWILIO_ACCOUNT_SID` | If SMS | twilio.com/console |
| `TWILIO_AUTH_TOKEN` | If SMS | twilio.com/console |
| `TWILIO_FROM_NUMBER` | If SMS | Your Twilio number in E.164 format e.g. `+13045550100` |
| `BRAND_CLUB_NAME` | — | Club name shown in emails and admin portal. Default: `Our Club` |
| `BRAND_TAGLINE` | — | Optional subtitle shown under club name in emails |
| `BRAND_HEADER_COLOR` | — | Email header background color (no `#`). Default: `1a56db` |
| `BRAND_ACCENT_COLOR` | — | PIN box and button color (no `#`). Default: `1a56db` |
| `BRAND_WEBSITE` | — | Shown in email footer |
| `BRAND_PHONE` | — | Shown in email footer |
| `BRAND_ADDRESS` | — | Shown in email footer |
| `ADMIN_PORT` | — | Port for admin portal. Default: `3000`. Set `PORT=3000` too for Railway. |
| `ADMIN_SECRET` | ✅ | Password to log into admin portal |
| `NOTIFY_MINUTES_BEFORE` | — | Minutes before reservation to send PIN. Default: `60` |
| `ACCESS_BUFFER_MINUTES` | — | Minutes before reservation the PIN becomes active. Default: `30` |
| `CLEANUP_BUFFER_MINUTES` | — | Minutes after reservation end before revoking access. Default: `15` |
| `TZ` | ✅ on Railway | Your timezone e.g. `America/New_York`. Critical — Railway runs in UTC. |
| `STATE_FILE` | — | Path for state persistence. Use `/tmp/state.json` on Railway. |

---

## Admin Portal

Once deployed, bookmark `https://YOUR_RAILWAY_URL/admin` on your phone.

**Logging in:** Enter your `ADMIN_SECRET` password once — the browser saves a session so you stay logged in.

**Dashboard shows:**
- Member name, email, and phone (tappable to call)
- Court name and reservation time
- PIN displayed prominently
- **Resend button** — one tap resends the email and SMS to the member instantly

**Search bar** filters across name, email, court, PIN, and reservation ID in real time.

**When a member can't get in:**
1. Open the bookmarked admin portal
2. Search by member name
3. Read out the PIN, or tap **Resend**

**Note on PINs:** Once a PIN is assigned in UniFi, the API only returns a hash — the actual digits cannot be retrieved. This integration saves the PIN at creation time in both the Railway logs and `state.json` so admins can always look it up.

---

## Finding your UniFi door/door-group IDs

```bash
curl -sk \
  -H "Authorization: Bearer YOUR_TOKEN" \
  "https://YOUR_UNIFI_IP:12445/api/v1/developer/door_groups/topology" \
  | python3 -m json.tool
```

Or find them in the UniFi Access UI — the door group ID appears in the URL when you click on a group.

---

## Setting up Resend

1. Sign up at **resend.com**
2. Go to **Domains → Add Domain** and enter your domain
3. Add the DNS records Resend provides into your DNS provider (Bluehost → Domains → DNS → Zone Editor)
4. Once verified, send from any address at your domain
5. Go to **API Keys → Create API Key** and copy it into `RESEND_API_KEY`

For testing before your domain is verified, set `EMAIL_FROM=onboarding@resend.dev`.

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| `Missing required environment variables` | Check all required variables are set in Railway |
| `CourtReserve API error: Unauthorized` | Check `CR_ORG_ID` and `CR_API_KEY` |
| `CourtReserve API error: Forbidden` | Ensure **ReservationReport (Read)** role is granted |
| `Fetched 0 reservations` | Confirm reservation exists and is confirmed in CourtReserve for today |
| `PIN generation failed` | UniFi token missing `view:credential` scope |
| `Create visitor failed` | UniFi token missing `edit:visitor` scope, or invalid `UNIFI_RESOURCES` IDs |
| `timeout of 15000ms exceeded` on UniFi | Console unreachable — check port forwarding and UniFi firewall rules |
| Email not sending | Check `RESEND_API_KEY` and domain verification in Resend dashboard |
| SMS not sending | Ensure `TWILIO_ENABLED=true` and all three Twilio vars are correct |
| Wrong times / missed reservations | Ensure `TZ=America/New_York` (or your timezone) is set in Railway |
| Admin portal not accessible | Ensure `PORT=3000` and `ADMIN_PORT=3000` are set, and domain is generated in Railway Settings → Networking |
| Duplicate PIN emails | State file was reset — harmless, member gets an extra notification |
| Visitor not deleted after reservation | Normal — deletion runs `CLEANUP_BUFFER_MINUTES` after end time |

---

## Tech Stack

| Layer | Technology |
|---|---|
| Reservation data | [CourtReserve](https://courtreserve.com) API |
| Door access | [UniFi Access](https://ui.com/door-access) API |
| Email delivery | [Resend](https://resend.com) |
| SMS delivery | [Twilio](https://twilio.com) (optional) |
| Hosting | [Railway](https://railway.app) (recommended) |
| Runtime | Node.js 18+ |

---

## Contributing

Contributions are welcome! See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines and a list of ideas for future improvements.

---

## License

MIT — see [LICENSE](LICENSE) for details.

---

## Security Notes

- Rotate `UNIFI_API_TOKEN` and `CR_API_KEY` periodically, and immediately if ever exposed
- Use a strong random string for `ADMIN_SECRET` — it protects all active PINs
- UniFi's self-signed TLS certificate is intentionally bypassed (per Ubiquiti's own API documentation)
- PINs are stored in plaintext in `state.json` and Railway logs for admin recovery — restrict Railway project access to trusted team members
- The admin portal uses HttpOnly session cookies — no credentials are stored in the browser
