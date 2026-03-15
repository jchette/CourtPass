# CourtPin

![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)
![Node.js](https://img.shields.io/badge/Node.js-18%2B-green.svg)
![Platform: Railway](https://img.shields.io/badge/Deploy-Railway-blueviolet.svg)

**Automated building access for court sports clubs.**

CourtPin connects [CourtReserve](https://courtreserve.com) with [UniFi Access](https://ui.com/door-access) to automatically send members a door PIN before their reservation — no staff involvement required. When a member books a court, CourtPin creates a time-limited visitor in UniFi Access, generates a unique PIN, and delivers it by email and optional SMS. After the reservation ends, access is automatically revoked.

Built for tennis, pickleball, squash, and racquetball clubs running CourtReserve for reservations and UniFi Access for door control.

---

## How it works

```
Every minute:
  ├─ Fetch today's active reservations from CourtReserve
  ├─ For each reservation within the notification window (default 60 min):
  │    ├─ Create a UniFi Visitor (access starts ACCESS_BUFFER_MINUTES early)
  │    ├─ Generate a random PIN
  │    ├─ Assign PIN to Visitor
  │    ├─ Send branded PIN email via Resend or SMTP
  │    └─ Optionally send PIN SMS via Twilio
  └─ Delete expired Visitors after reservations end
```

Multi-player reservations are fully supported — every player gets their own Visitor record and unique PIN.

---

## Example timeline

With defaults (`NOTIFY_MINUTES_BEFORE=60`, `ACCESS_BUFFER_MINUTES=30`):

```
11:00 AM   CourtPin detects a 12:00 PM reservation
            Creates UniFi Visitor (active 11:30 AM to 1:00 PM)
            Sends email: "Your Club Access PIN - Sun Mar 15, 12:00 PM"

11:30 AM   PIN becomes active — member can enter the building

12:00 PM   Reservation starts

 1:00 PM   Reservation ends

 1:15 PM   Visitor deleted, PIN revoked
```

---

## Requirements

| Requirement | Notes |
|---|---|
| Node.js 18+ | nodejs.org |
| CourtReserve with API access | Requires ReservationReport Read permission |
| UniFi Access 1.9.1+ | API token needs view:credential and edit:visitor scopes |
| Resend or SMTP | Email delivery — see docs/email-setup.md |
| Twilio | Optional SMS — free trial at twilio.com |

---

## Quick start

```bash
git clone https://github.com/jchette/courtpin.git
cd courtpin
npm install
cp env.example .env
# Edit .env with your credentials
npm start
```

---

## Documentation

All detailed guides live in the `docs/` folder:

| Document | What it covers |
|---|---|
| [docs/configuration.md](docs/configuration.md) | Every environment variable explained, with a Railway copy-paste block |
| [docs/hosting.md](docs/hosting.md) | All six hosting options: Railway, Raspberry Pi, Docker, NAS, VPS, Windows |
| [docs/unifi-setup.md](docs/unifi-setup.md) | API token creation, door group IDs, port forwarding, Cloudflare Tunnel |
| [docs/email-setup.md](docs/email-setup.md) | Resend setup, SMTP setup, Gmail app passwords, local SMTP servers |
| [docs/admin-portal.md](docs/admin-portal.md) | Using the mobile-friendly PIN lookup and resend portal |
| [docs/troubleshooting.md](docs/troubleshooting.md) | Every known issue with symptoms, causes, and fixes |

---

## Hosting options summary

CourtPin runs on any always-on machine or cloud platform:

| Method | Cost | Best for |
|---|---|---|
| Railway | Free to $5/mo | No local hardware — easiest setup |
| Raspberry Pi | Free (hardware ~$50) | Local hosting, no port forwarding needed |
| Docker | Free | NAS, home server, or existing Docker setup |
| Synology / QNAP NAS | Free | Clubs with a NAS already running |
| VPS | $4 to $6/mo | Cloud hosting with full control |
| Windows PC | Free | Always-on PC at the facility |

**Local hosting advantage:** When CourtPin runs on the same network as your UniFi console, no port forwarding is needed — use the local IP directly.

Full details: [docs/hosting.md](docs/hosting.md)

---

## Tech stack

| Layer | Technology |
|---|---|
| Reservation data | CourtReserve API |
| Door access | UniFi Access API |
| Email (cloud) | Resend |
| Email (local) | SMTP via nodemailer |
| SMS | Twilio (optional) |
| Hosting | Railway (recommended) |
| Runtime | Node.js 18+ |

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines, development setup, and ideas for future improvements.

---

## License

MIT — see [LICENSE](LICENSE). Free to use, modify, and distribute.
