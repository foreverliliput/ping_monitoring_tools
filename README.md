# ping_monitoring_tools
A simple web-based MikroTik ping monitoring tool using the MikroTik API. Monitor multiple MikroTik devices, run ping tests at the same time, check latency and packet loss, and easily add, edit, or remove devices. Built for network admins, ISPs, and MikroTik users.

## Run it

```bash
npm install
cp .env.example .env      # edit ADMIN_PASSWORD at minimum
npm start
```

Open **http://localhost:4000**, log in with the password from `.env`.
That's it — start adding devices and it monitors them immediately.


## Requirements

- Node.js 18+
- The system `ping` command (already on every Linux/macOS/Windows install by
  default — nothing to configure, no root/admin privileges needed, unlike
  raw-socket ping engines).
- The system `traceroute` command (Linux/macOS: usually pre-installed,
  otherwise `sudo apt install traceroute` or `brew install traceroute`;
  Windows: built-in `tracert` is used automatically). The Traceroute button
  in the UI will just show a clear error if this isn't installed — nothing
  else in the app depends on it.

## How storage works

Everything you create (devices, profiles, alert rules, alert history,
MikroTik router credentials) lives in one file: `data/db.json`. It's loaded
into memory when the app starts and saved automatically whenever you change
something. Back it up by copying that one file; reset by deleting it.

Live ping stats (current status, latency, min/avg/max, uptime %) are **not**
in that file — they're rebuilt in memory as checks run, which is why
restarting the app shows a brief "gathering data" period before the grid
fills back in.

## What's included

- **Services tab**: spreadsheet-style grid — sort any column, search, filter
  by category/POP/status/server, group by POP or category, pin important
  rows, select multiple rows for bulk edit, CSV import/export. Real-time
  updates over WebSocket (no page refresh). Columns: Service Name (with an
  inline online/offline dot), IP, Category, Server, Latency, Loss %, Sent,
  Received, Avg, Jitter, Notes, Actions.
  - **Loss %** is a true cumulative sent/receive ratio (e.g. 100 sent, 99
    received → 1%), not a snapshot of the single latest check.
  - **Refresh**: click ↻ on any row, or "Refresh All" in the toolbar, to
    wipe that service's (or every service's) accumulated stats and start a
    clean report from the next check onward.
- **Monitor Via**: each service can be monitored either directly from this
  server (default) or relayed through one of your saved MikroTik routers'
  own ping tool - useful for seeing the path from the router's actual
  vantage point rather than from wherever this app happens to run. Set this
  per-service in the Add/Edit form's "Monitor Via" dropdown; the grid's
  **Server** column and the toolbar's **Server** filter both reflect it.
- **Traceroute**: click "Trace" on any service row for a live hop-by-hop
  run (uses ICMP-mode probes, since UDP-mode traceroute is commonly blocked
  inside ISP networks).
- **Profiles tab**: save named device groups (e.g. "Core Network").
- **Alert Rules tab**: rules for offline duration, latency, packet loss, and
  jitter, each with a consecutive-checks requirement so a single blip
  doesn't fire an alert. Existing rules can be fully edited (name, metric,
  operator, threshold, consecutive checks, target device, Telegram
  toggle, enabled/disabled) via the Edit button, not just toggled on/off.
  Alerts stay visible until you explicitly remove them (they don't silently
  disappear) - open the 🔔 panel to Acknowledge or Remove.
- **Settings tab**: connect a Telegram bot directly in the UI - paste the
  bot token (from [@BotFather](https://t.me/BotFather)) and your chat ID,
  hit Save, then "Send Test Message" to confirm it actually reaches your
  chat before relying on it for alerts. No more editing `.env` and
  restarting the app to change this. `.env` values still work as a fallback
  if the Settings tab is left blank.
- **MikroTik tab**: add routers, test the connection, pull identity/
  version/CPU/memory/uptime info via the RouterOS API, and assign them to
  services under "Monitor Via" in the Add/Edit Service form.

## Notes on scale and security (read before exposing this beyond localhost)

- **Ping engine**: uses the OS `ping` command (via the `ping` npm package),
  not raw ICMP sockets. This is what makes "no privilege setup" possible,
  but it means each check spawns a short-lived process — comfortable for
  tens to a few hundred devices, not thousands at sub-second intervals. If
  you outgrow this, `lib/ping.js` is the one file to swap for a raw-socket
  engine (e.g. `net-ping`); nothing else in the app needs to change.
- **MikroTik-relayed monitoring**: each check opens a short-lived RouterOS
  API connection to run one ping via `/ping`. This is noticeably heavier
  than a local ICMP check, so router-monitored services default to a 15s
  interval (vs. 10s for local) and are capped at a lower concurrency
  (`MIKROTIK_MAX_CONCURRENT`, default 10) - both tunable in `.env`. Fine for
  tens of router-relayed services; if you need hundreds on tight intervals,
  `lib/mikrotik.js`'s `checkViaRouter()` would benefit from a persistent
  pooled connection per router instead of reconnecting every check.
- **Single admin login**: one password from `.env`, no per-user accounts or
  roles. Fine for a small team sharing one login; if you need individual
  accounts/audit trail later, that's a `lib/auth.js` upgrade, not a rewrite.
- **MikroTik passwords** are stored in `data/db.json` as plain text to keep
  this dependency-free. Don't commit that file, and restrict its file
  permissions if this runs on a shared machine.
- **CSV import** does no escaping beyond basic comma-splitting — fine for
  the simple `name,ipAddress,category,pop,vendor,notes` format described in
  the UI, not a full RFC-4180 CSV parser. (POP/Vendor are no longer offered
  in the interactive Add Service form, but CSV import still accepts them if
  you're bulk-importing from an existing inventory.)

## Project layout

```
server.js          Express app + all API routes, one file for easy reading
lib/
  db.js             The entire "database" - one JSON file, load/save
  ping.js           Ping engine (system ping command)
  scheduler.js      Concurrency-limited tick loop that fires due checks
  stats.js          In-memory live stats + short rolling history
  traceroute.js     Wraps system traceroute/tracert
  alerts.js         Rule evaluation + Telegram notifications
  mikrotik.js       RouterOS API client wrapper
  auth.js           Single-admin login/session tokens
  ws.js             WebSocket broadcast to the dashboard
public/
  index.html        Dashboard shell
  app.js            All frontend logic, vanilla JS, no build step
  styles.css        Dark NOC theme
data/
  db.json           Created automatically on first run
```

## Next steps

- Native PDF/Excel reports (currently CSV only — see `/api/reports/alert-log.csv`)
- Multi-location comparison (a second instance posting into a shared view)
- Per-user accounts/roles if you outgrow the single shared login
