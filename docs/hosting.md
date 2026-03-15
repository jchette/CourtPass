# Hosting Options

CourtPin is a long-running background process and needs a host that keeps it alive 24/7. Regular shared web hosting (like Bluehost) is not suitable. Below are all supported options from simplest to most advanced.

---

## Choosing a hosting method

| Method | Monthly cost | Technical level | Best for |
|---|---|---|---|
| Railway | Free to $5 | Beginner | Clubs with no local server hardware |
| Raspberry Pi | Free (hardware ~$50 one-time) | Beginner | Clubs wanting local hosting on cheap hardware |
| Docker | Free | Intermediate | Clubs with a NAS, home server, or existing Docker setup |
| Synology / QNAP NAS | Free | Beginner | Clubs with a NAS already running |
| VPS | $4–$6/mo | Intermediate | Cloud hosting with full control, no Railway dependency |
| Windows PC | Free | Beginner | Clubs with an always-on Windows PC at the facility |

**Key advantage of local hosting (Pi, NAS, Docker, Windows):** CourtPin runs on the same network as your UniFi console so no port forwarding is needed. Set `UNIFI_HOST` to the local IP directly (e.g. `https://192.168.1.1:12445`) and skip the entire UniFi networking setup.

---

## Option 1 — Railway (recommended for most clubs)

Railway is a cloud platform that keeps CourtPin running with zero server management. It deploys automatically from GitHub on every push.

**Prerequisites:** A GitHub account and a Railway account (sign up with GitHub at railway.app).

### Step 1 — Push code to GitHub

1. Create a new **private** repo on github.com named `courtpin`
2. Upload all CourtPin files (do NOT upload your `.env` file — use Railway's Variables tab for credentials)
3. Commit

### Step 2 — Create Railway project

1. Go to railway.app → **New Project → Deploy from GitHub repo**
2. Select your `courtpin` repo
3. Railway starts a deployment — it will fail until you add variables. That is expected.

### Step 3 — Add environment variables

1. Click your service → **Variables → Raw Editor**
2. Paste the block from [configuration.md](configuration.md) and fill in your values
3. Click **Update Variables** — Railway redeploys automatically

### Step 4 — Generate a public URL

1. Click your service → **Settings → Networking → Generate Domain**
2. Bookmark `https://YOUR_RAILWAY_URL/admin` on your phone

### Step 5 — Verify

Click **Deployments → latest → View Logs**. You should see:
```
[INFO] CourtReserve <-> UniFi Access integration starting
[INFO] Fetched N reservation(s)
[INFO] Scheduler running — checking every minute.
```

### Step 6 — Test

Temporarily set `NOTIFY_MINUTES_BEFORE=1440`, create a test reservation in CourtReserve, and watch the logs. Change it back to `60` once confirmed working.

---

## Option 2 — Raspberry Pi (or any always-on Linux machine)

Any Raspberry Pi 3 or newer handles CourtPin easily. Any other always-on Linux machine works identically.

### Step 1 — Install Node.js

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
node --version   # should show v20.x.x or later
```

### Step 2 — Get the code

```bash
git clone https://github.com/YOUR_USERNAME/courtpin.git
cd courtpin
npm install
```

### Step 3 — Configure

```bash
cp env.example .env
nano .env
```

Since the Pi is on the same local network as your UniFi console:
```
UNIFI_HOST=https://192.168.1.1:12445
```

No port forwarding needed. Save with Ctrl+O, exit with Ctrl+X.

### Step 4 — Test

```bash
npm start
# Press Ctrl+C to stop once confirmed working
```

### Step 5 — Keep it running permanently with PM2

```bash
npm install -g pm2
pm2 start index.js --name courtpin
pm2 save
pm2 startup
# Run the command printed by pm2 startup to enable auto-start on reboot
```

**Useful PM2 commands:**
```bash
pm2 status              # check if running
pm2 logs courtpin      # view live logs
pm2 restart courtpin   # restart after editing .env
pm2 stop courtpin      # stop the service
```

---

## Option 3 — Docker

Docker packages CourtPin into a self-contained image that runs identically on any machine. CourtPin includes a `Dockerfile` and `docker-compose.yml` ready to use.

### Install Docker

On Ubuntu / Raspberry Pi OS / Debian:
```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
# Log out and back in for the group change to take effect
```

On other systems see [docs.docker.com/get-docker](https://docs.docker.com/get-docker).

### Option A — Docker Compose (recommended)

```bash
# 1. Configure
cp env.example .env
nano .env   # fill in your values

# 2. Start
docker compose up -d

# 3. View logs
docker compose logs -f

# 4. Stop
docker compose down

# 5. Restart after changing .env
docker compose down && docker compose up -d
```

CourtPin automatically restarts on crash or reboot due to `restart: unless-stopped` in the compose file.

### Option B — Plain Docker run

```bash
# Build
docker build -t courtpin .

# Run
docker run -d \
  --name courtpin \
  --restart unless-stopped \
  --env-file .env \
  -v courtpin_data:/data \
  -p 3000:3000 \
  courtpin

# View logs
docker logs -f courtpin
```

### Updating

```bash
git pull
docker compose down
docker compose up -d --build
```

---

## Option 4 — Synology or QNAP NAS

If your facility already has a NAS running, it can host CourtPin with no additional hardware.

### Synology NAS

1. Open **Package Center** → install **Container Manager**
2. Open Container Manager → **Project → Create**
3. Paste the contents of `docker-compose.yml`
4. Set environment variables in the UI or reference your `.env` file
5. Click **Build** then **Start**

Admin portal accessible at `http://YOUR_NAS_IP:3000/admin` on your local network.

### QNAP NAS

1. Open **App Center** → install **Container Station**
2. Open Container Station → **Create → Create Application**
3. Paste the contents of `docker-compose.yml`
4. Set your environment variables
5. Click **Create**

---

## Option 5 — VPS

A virtual private server gives you cloud hosting with full control and no Railway dependency. The state file is stable across restarts and you can run other services on the same machine.

**Recommended providers:**

| Provider | Monthly cost | Notes |
|---|---|---|
| Hetzner | ~$4 | Best value, European data centers |
| DigitalOcean | $6 | Beginner friendly, excellent documentation |
| Vultr | $6 | Good global coverage |
| Linode (Akamai) | $5 | Reliable, long established |

Choose the smallest plan with 1 GB RAM — CourtPin uses very little memory.

### Setup

1. Create an Ubuntu 22.04 or 24.04 server
2. SSH in: `ssh root@YOUR_SERVER_IP`
3. Follow the same steps as Option 2 (Raspberry Pi) from Step 1 onward

Since a VPS is in the cloud you still need port forwarding or a Cloudflare Tunnel to reach your UniFi console. See [unifi-setup.md](unifi-setup.md).

---

## Option 6 — Windows PC

If there is a Windows PC at the facility that is always powered on, it can run CourtPin.

### Step 1 — Install Node.js

1. Go to [nodejs.org](https://nodejs.org) and download the **LTS** installer
2. Run the installer with default settings
3. Open **Command Prompt** and verify: `node --version`

### Step 2 — Get the code

1. Go to your GitHub repo → **Code → Download ZIP**
2. Extract to `C:\courtpin`
3. Open Command Prompt: `cd C:\courtpin`

### Step 3 — Install and configure

```cmd
npm install
copy env.example .env
notepad .env
```

Fill in all values and save.

### Step 4 — Test

```cmd
npm start
```

### Step 5 — Keep it running permanently

```cmd
npm install -g pm2
npm install -g pm2-windows-startup
pm2 start index.js --name courtpin
pm2 save
pm2-startup install
```

**Useful commands:**
```cmd
pm2 status
pm2 logs courtpin
pm2 restart courtpin
```
