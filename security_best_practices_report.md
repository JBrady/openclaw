# OpenClaw Security Best Practices Report

## Executive Summary
Your OpenClaw installation is functioning and uses pairing controls, and the gateway ports are bound to `127.0.0.1`, which limits exposure. The main security risks are **secrets stored in local config files** and **token-based access that could be reused if leaked**. I recommend tightening secret handling and clarifying network exposure settings while keeping current behavior intact.

## Findings

### [MED-1] Long‑lived secrets stored in plaintext config files
**Severity:** Medium

**Location:**
- `.env` lines 7, 13 (`OPENCLAW_GATEWAY_TOKEN`, `TELEGRAM_BOT_TOKEN`)  
  `.env:7`, `.env:13`
- `~/.openclaw/openclaw.json` lines 49–50, 58–68 (Telegram bot token, gateway tokens)
  `~/.openclaw/openclaw.json:49`, `~/.openclaw/openclaw.json:58`, `~/.openclaw/openclaw.json:66`

**Evidence:**
- `.env` stores tokens in plaintext.
- `~/.openclaw/openclaw.json` stores the same tokens and gateway auth token.

**Impact:**
If these files are copied, backed up insecurely, or exposed to other local users/processes, an attacker can control the gateway or the Telegram bot.

**Fix (safe, minimal):**
1. Keep `.env` with `0600` permissions (already correct) and avoid storing tokens in multiple places. Prefer **env-only** secrets and remove `botToken` from the JSON where possible.
2. Rotate tokens periodically (gateway token + Telegram bot token) and update `.env` after rotation.
3. Avoid sharing the Control UI token URL; rotate the gateway token if it’s ever shared.

**Mitigation:**
- Confirm `.env` and `~/.openclaw/openclaw.json` are owned by your user and are `600`.
- Ensure `.env` is not committed (it is in `.gitignore`, so this is already good).

**Notes:** This is a common and acceptable local-dev setup, but it’s the top risk if the machine is shared or backups are not private.

---

### [LOW-1] Gateway bind mode is `lan` (exposure depends on port publish)
**Severity:** Low

**Location:**
- `.env:12` (`OPENCLAW_GATEWAY_BIND=lan`)
- `docker-compose.yml:20` (gateway started with `--bind ${OPENCLAW_GATEWAY_BIND:-lan}`)

**Evidence:**
- Binding is set to `lan`, which allows LAN interfaces inside the container.
- Ports are published only to `127.0.0.1`, which **does** limit host exposure.

**Impact:**
This is safe **as configured**, but if the port mapping is ever changed to `0.0.0.0` or the host firewall changes, the gateway could become reachable on the LAN without TLS.

**Fix (safe, minimal):**
- If you only use local access, set `OPENCLAW_GATEWAY_BIND=loopback` in `.env`.
- If you need remote access, prefer Tailscale/HTTPS rather than exposing raw ports.

**Mitigation:**
- Keep `ports` in `docker-compose.yml` bound to `127.0.0.1`.

---

### [LOW-2] Device pairing approvals are persistent and local
**Severity:** Low

**Location:**
- `~/.openclaw/devices/paired.json` (paired device list)

**Evidence:**
- Approved device entries persist, enabling continued CLI/control UI access.

**Impact:**
If a machine is shared or compromised, a previously paired device may remain valid longer than desired.

**Fix (safe, minimal):**
- Periodically review and remove old devices from `paired.json`.
- Rotate the gateway token after removing devices if you suspect compromise.

**Mitigation:**
- Keep `~/.openclaw` permissions restricted to your user.

---

## Positive Security Posture
- Gateway ports are bound to `127.0.0.1` in `docker-compose.yml`, limiting exposure.
- Pairing is enabled for Telegram DMs and the Control UI (pairing required is enforced).
- `.env` file permissions are `600`, which is appropriate for local secrets.

## Recommended Next Steps (Non‑breaking)
1. Decide whether you want LAN binding. If not, set `OPENCLAW_GATEWAY_BIND=loopback`.
2. Rotate the gateway token and Telegram bot token and update `.env` accordingly.
3. Review `~/.openclaw/devices/paired.json` and remove any stale devices.

If you want, I can implement the safe, non‑breaking changes (e.g., switching to loopback bind, pruning paired devices) and verify the gateway still runs.
