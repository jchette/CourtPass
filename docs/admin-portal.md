# Admin Portal

CourtPin includes a built-in mobile-friendly web portal for looking up PINs and resending them to members. No separate app or login to Railway or UniFi is needed.

---

## Accessing the portal

Your portal URL is:
```
https://YOUR_RAILWAY_URL/admin
```

Or if running locally:
```
http://YOUR_HOST_IP:3000/admin
```

**Bookmark this on your phone.** It is the page you will open whenever a member cannot get in.

---

## Logging in

Enter your `ADMIN_SECRET` password on the login screen. Your browser saves a session cookie so you stay logged in — you only need to enter the password once per device.

Tap **Log out** in the top right corner when done on a shared device.

---

## The dashboard

The dashboard shows one card per active reservation today, sorted by start time. Each card displays:

| Field | Description |
|---|---|
| **Member name** | From the CourtReserve account |
| **Time range badge** | Start–end time (e.g. `12:00 PM–1:00 PM`) |
| **Email** | Where the PIN was sent |
| **Phone** | Tappable — opens your phone's dialer to call directly |
| **Court** | Which court was booked |
| **Date** | The reservation date |
| **Reservation ID** | CourtReserve reference number |
| **Access PIN** | Displayed in large digits |
| **Resend button** | Resends email and SMS instantly |

---

## Searching

Type in the search bar at the top to filter cards in real time. You can search by:

- Member name
- Email address
- Phone number
- Court name
- PIN digits
- Reservation ID

The badge in the header updates to show how many results match.

---

## Resending a PIN

When a member cannot get in because they did not receive their email, deleted it, or cannot find it:

1. Open the admin portal on your phone
2. Search for the member by name
3. Find their card
4. Tap **Resend**
5. CourtPin immediately resends the email (and SMS if enabled)
6. A confirmation appears on the card: `✅ Resent to member@email.com`

The same PIN is resent — a new one is not generated. The member's access window in UniFi is unchanged.

---

## Why the PIN is visible here

UniFi Access is designed so that once a PIN is assigned to a Visitor, the API only returns a cryptographic hash — the actual digits cannot be retrieved later. CourtPin saves the PIN at the moment of creation in both the Railway logs and the `state.json` state file. The admin portal reads from the state file to display PINs here.

This is why it is important to:
- Keep `ADMIN_SECRET` strong and private
- Restrict Railway project access to trusted team members
- Use the **Log out** button on shared devices

---

## If a PIN does not appear

Cards may show "PIN not stored for this record" if the reservation was processed before PIN logging was added in an earlier version. In this case:

1. Check the Railway logs and search for the reservation ID — the PIN is logged at processing time
2. Or look up the member in UniFi Access → Visitors — the Visitor record will still be there but the PIN digits are not visible in UniFi (only the hash)
3. As a last resort, delete the entry from `state.json` (or change `STATE_FILE` to force reprocessing) and let CourtPin reprocess the reservation — it will generate a new PIN and send a new email

---

## Setting up the admin portal on Railway

1. Make sure `PORT=3000` and `ADMIN_PORT=3000` are both set in Railway Variables
2. Go to Railway → your service → **Settings → Networking**
3. Click **Generate Domain**
4. Your URL will be something like `courtpin-production.up.railway.app`
5. Bookmark `https://courtpin-production.up.railway.app/admin`

---

## Security

- The admin portal is protected by an `HttpOnly` session cookie — your password is never stored in the browser, only a session token
- Sessions exist only in memory — they are cleared if CourtPin restarts
- All active PINs are visible to anyone who can log in — keep `ADMIN_SECRET` strong
- Use the Log out button on any device you do not own
