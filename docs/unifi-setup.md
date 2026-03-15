# UniFi Access Setup

This guide covers creating the API token, finding your door group IDs, and making your UniFi console reachable from the internet (required for cloud hosting on Railway or a VPS).

> **Local hosting shortcut:** If CourtPin runs on the same network as your UniFi console (Raspberry Pi, NAS, Docker, Windows PC), skip the remote access section entirely. Just set `UNIFI_HOST=https://192.168.1.1:12445` using the local IP.

---

## Creating the API token

CourtPin needs an API token with two permission scopes:

- `view:credential` — generates PIN codes
- `edit:visitor` — creates, updates, and deletes Visitor records

**Steps:**

1. Log into [unifi.ui.com](https://unifi.ui.com)
2. Click on your console
3. Go to **Access → Settings → General → Advanced**
4. Scroll to **API Token** and click **Create New**
5. Name it `CourtPin`
6. Set validity to the longest available or no expiry
7. Check both `view:credential` and `edit:visitor`
8. Click **Create**
9. **Copy the token immediately** — it is only shown once

Set this as `UNIFI_API_TOKEN` in your environment variables.

---

## Finding your door group IDs

### Method A — From the UniFi UI (easiest)

1. Log into your UniFi console locally or via unifi.ui.com
2. Open the **Access** application
3. Go to **Settings → Door Groups** (or **Spaces**)
4. Click the door group you want members to access
5. Look at the browser URL — the ID is the long string at the end:
   ```
   https://192.168.1.1/access/settings/door-groups/9bee6e0e-108d-4c52-9107-76f2c7dea4f1
   ```
   The ID is `9bee6e0e-108d-4c52-9107-76f2c7dea4f1`

### Method B — Via the API

From any computer on the same local network, run this in Terminal (Mac) or Command Prompt (Windows):

```bash
curl -sk \
  -H "Authorization: Bearer YOUR_UNIFI_API_TOKEN" \
  "https://192.168.1.1:12445/api/v1/developer/door_groups/topology" \
  | python3 -m json.tool
```

Output example:
```json
{
  "code": "SUCCESS",
  "data": [
    {
      "id": "9bee6e0e-108d-4c52-9107-76f2c7dea4f1",
      "name": "Main Building",
      "type": "door_group",
      "doors": [
        {
          "id": "6ff875d2-af87-470b-9cb5-774c6596afc8",
          "name": "Front Door"
        }
      ]
    }
  ]
}
```

### Setting UNIFI_RESOURCES

```bash
# A whole door group (all doors in the group):
UNIFI_RESOURCES=door_group:9bee6e0e-108d-4c52-9107-76f2c7dea4f1

# A single specific door:
UNIFI_RESOURCES=door:6ff875d2-af87-470b-9cb5-774c6596afc8

# Multiple entries (comma-separated, no spaces):
UNIFI_RESOURCES=door_group:9bee6e0e-108d-4c52-9107-76f2c7dea4f1,door:6ff875d2-af87-470b-9cb5-774c6596afc8

# Empty (test CourtPin without granting door access):
UNIFI_RESOURCES=
```

---

## Making UniFi reachable from the internet

Required when hosting on Railway or a VPS. Skip this section if hosting locally.

### Option A — Port forwarding + DuckDNS

Port forwarding routes incoming internet traffic on port 12445 to your UniFi console.

#### Step 1 — Find your UniFi console's local IP

If your UniFi gateway (UDM, UCG, etc.) is also your Access controller, the IP is usually `192.168.1.1`. Confirm by opening `https://192.168.1.1` in a browser on your local network.

If Access runs on a separate device: UniFi Network → Devices → click the device → find its IP.

#### Step 2 — Set up port forwarding in UniFi

1. Go to **UniFi Network → Firewall & Security → Port Forwarding**
2. Click **Create Entry**:
   - Name: `CourtPin API`
   - Protocol: TCP
   - WAN Interface: Internet 1
   - WAN Port: `12445`
   - Forward IP: your console's local IP
   - Forward Port: `12445`
3. Save

#### Step 3 — Add a firewall rule

1. Go to **Firewall & Security → Firewall Rules → WAN**
2. Click **Create Entry**:
   - Name: `CourtPin Allow`
   - Action: Accept
   - Type: **Internet Local** if Access is on the gateway itself, or **Internet In** if on a separate device
   - Protocol: TCP
   - Destination IP: your console's local IP
   - Destination Port: `12445`
3. Save

#### Step 4 — Set up DuckDNS (permanent hostname)

Most ISPs change your public IP periodically. DuckDNS gives you a permanent hostname for free.

1. Go to [duckdns.org](https://duckdns.org) and sign in with Google
2. Enter a subdomain (e.g. `yourclub`) and click **Add Domain**
3. Install the DuckDNS updater to keep it pointing to your current IP:
   - Many Asus, Netgear, and UniFi routers support DuckDNS under DDNS settings — enter your token and hostname there
   - On Windows or Linux, use the install script from the DuckDNS install page
4. Set in Railway: `UNIFI_HOST=https://yourclub.duckdns.org:12445`

#### Step 5 — Verify

Go to [portchecker.co](https://portchecker.co), enter your public IP and port `12445`. It should show **Open**. If it shows Closed, recheck your port forwarding and firewall rules.

---

### Option B — Cloudflare Tunnel (no open ports, more secure)

A Cloudflare Tunnel creates an outbound-only encrypted connection from your local network to Cloudflare's edge. No ports are opened on your router.

#### Step 1 — Create a Cloudflare account

Go to [cloudflare.com](https://cloudflare.com) and sign up free. No domain needed.

#### Step 2 — Set up Zero Trust

1. From the Cloudflare dashboard, click **Zero Trust**
2. Choose a team name (anything works)
3. Select the **Free plan**

#### Step 3 — Create a tunnel

1. Go to **Networks → Tunnels → Create a Tunnel**
2. Select **Cloudflared** → Next
3. Name it `unifi-access` → Save Tunnel
4. Leave this page open — you need the install command shown

#### Step 4 — Install the cloudflared agent

On a Windows PC at your facility:
1. Download the installer shown on the Cloudflare page
2. Run it and execute the `cloudflared tunnel run --token YOUR_TOKEN` command

On Linux / Raspberry Pi:
```bash
curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64 -o cloudflared
chmod +x cloudflared
sudo mv cloudflared /usr/local/bin
cloudflared tunnel run --token YOUR_TOKEN_FROM_CLOUDFLARE
```

The tunnel status should change to **Connected** in Cloudflare.

#### Step 5 — Configure a public hostname

1. On your tunnel page click **Public Hostname → Add a public hostname**
2. Fill in:
   - Subdomain: `unifi`
   - Domain: the free `.cfargotunnel.com` option or your own domain
   - Type: HTTPS
   - URL: `192.168.1.1:12445`
3. Under **TLS Settings** check **No TLS Verify** — required because UniFi uses a self-signed certificate
4. Save

#### Step 6 — Update Railway

```
UNIFI_HOST=https://unifi.yoursubdomain.cfargotunnel.com
```

No port number needed — Cloudflare routes over standard HTTPS (port 443).

---

## API permissions reference

CourtPin uses four UniFi Access API endpoints:

| Method | Endpoint | Permission needed | Purpose |
|---|---|---|---|
| `POST` | `/api/v1/developer/visitors` | `edit:visitor` | Create Visitor record |
| `POST` | `/api/v1/developer/credentials/pin_codes` | `view:credential` | Generate random PIN |
| `PUT` | `/api/v1/developer/visitors/:id/pin_codes` | `edit:visitor` | Assign PIN to Visitor |
| `DELETE` | `/api/v1/developer/visitors/:id` | `edit:visitor` | Delete Visitor after reservation |

> **Note on PINs:** Once a PIN is assigned in UniFi, the API only returns a cryptographic hash — the actual digits cannot be retrieved. CourtPin saves the PIN at creation time in both Railway logs and the state file so admins can look it up via the admin portal.
