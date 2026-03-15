# Contributing to CourtPass

Thank you for your interest in contributing! CourtPass is a small open source project and all contributions are welcome — bug fixes, new features, documentation improvements, and real-world feedback from clubs using it.

---

## Ways to contribute

### 🐛 Report a bug
Open an issue and include:
- What you expected to happen
- What actually happened
- Relevant Railway log output
- Your UniFi Access version and CourtReserve plan

### 💡 Suggest a feature
Open an issue with the label `enhancement` and describe the use case. Real stories from club operators are especially helpful.

### 🔧 Submit a pull request
1. Fork the repo
2. Create a branch: `git checkout -b feature/your-feature-name`
3. Make your changes
4. Test it against a real CourtReserve + UniFi setup if possible
5. Open a pull request with a clear description of what changed and why

---

## Development setup

```bash
git clone https://github.com/YOUR_USERNAME/courtpass.git
cd courtpass
npm install
cp env.example .env
# Fill in your credentials
npm run dev   # uses nodemon for auto-reload
```

---

## Code style

- Plain Node.js — no TypeScript, no build step
- Keep dependencies minimal (currently: `axios`, `dotenv`, `node-cron`)
- All configuration via environment variables — no hardcoded values
- Functions should do one thing and be easy to read without comments

---

## Ideas for future contributions

These are things that would genuinely improve CourtPass for clubs:

- **Webhook support** — trigger on CourtReserve reservation creation instead of polling
- **Cancellation handling** — revoke PIN immediately when a reservation is cancelled
- **Multi-facility support** — different door groups per court type
- **Web-based setup wizard** — guided first-run configuration instead of manual env vars
- **Email template customization** — allow clubs to provide their own HTML template
- **Support for other access control systems** — Brivo, Salto, Kisi, etc.
- **Docker Compose file** — make local deployment easier

---

## Questions?

Open an issue — happy to help.
