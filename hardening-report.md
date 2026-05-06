# VPS Security Hardening Report

## Project: AI Assistant Platform on Ubuntu 24.04 VPS

**Server:** `<your-server-hostname>`  
**OS:** Ubuntu 24.04.4 LTS (Noble Numbat)  
**Kernel:** 6.8.0-110-generic (x64)  
**Hostname:** `<your-server-hostname>`  
**Provider:** `<your-vps-provider>`  
**Primary Use:** AI assistant platform (OpenClaw Gateway + Ollama)  
**Network:** Tailscale mesh VPN (`<your-tailscale-ip>`)

---

## Executive Summary

This VPS runs an AI assistant platform with a local LLM. Initial security audit revealed significant exposure — SSH open to the entire internet, no verified firewall enforcement, and weak brute-force protection. A phased hardening plan was executed to reduce the attack surface to near-zero for public-facing services, relying on Tailscale for authorized access.

---

## Initial State (2026-04-20 Audit)

| Finding | Severity | Status |
|---------|----------|--------|
| SSH listening on 0.0.0.0:22 (public internet) | CRITICAL | |
| SSH listening on [::]:22 (public IPv6) | CRITICAL | |
| No fail2ban installed | HIGH | |
| UFW installed but status unverified | HIGH | |
| PasswordAuthentication disabled (via drop-in) | — | ✅ Already good |
| PermitRootLogin no | — | ✅ Already good |
| Unattended upgrades active | — | ✅ Already good |
| Ollama bound to localhost only | — | ✅ Already good |
| Fail2ban ban time too short (10m) | MEDIUM | |
| Unnecessary services running (ModemManager, fwupd, etc.) | LOW | |
| MOTD leaks hosting provider | LOW | |

---

## Changes Applied

### Phase 1: Critical Fixes

#### Change 1 — SSH ListenAddress restricted to Tailscale (2026-04-21 ~21:20 CEST)

**Problem:** SSH daemon was accepting connections on all network interfaces (`0.0.0.0:22` and `[::]:22`), exposing login attempts to the entire internet.

**Action:**
- Edited `/etc/ssh/sshd_config`:
  - Uncommented `Port 22`
  - Set `ListenAddress <your-tailscale-ip>` (Tailscale IP)
  - Commented out `ListenAddress ::`
- Created backup: `/etc/ssh/sshd_config.bak.2026-04-21`

**Result:** Initial change had no effect — systemd socket activation (`ssh.socket`) was independently binding port 22 on all interfaces, overriding sshd_config.

#### Change 2 — Disabled ssh.socket, switched to ssh.service only (2026-04-21 ~21:29 CEST)

**Problem:** Ubuntu 24.04 uses systemd socket activation for SSH by default. The `ssh.socket` unit binds port 22 on all interfaces regardless of `sshd_config` `ListenAddress` directives.

**Action:**
```bash
sudo systemctl disable --now ssh.socket
sudo systemctl daemon-reload
sudo systemctl restart ssh.service
```

**Result:** SSH now listens exclusively on `<your-tailscale-ip>:22` (Tailscale interface only).

**Verification:**
```
$ ss -tlnp | grep :22
LISTEN 0 128 <your-tailscale-ip>:22 0.0.0.0:*
```

**Before:** `0.0.0.0:22` + `[::]:22` → **After:** `<your-tailscale-ip>:22`

**Security Impact:** SSH is completely unreachable from the public internet. Tailscale connectivity is now required for all SSH access. This eliminates the primary attack vector for brute-force and exploit-based intrusions.

**Key Lesson:** Ubuntu 24.04 uses `ssh.socket` for binding — disabling it is required for `ListenAddress` in `sshd_config` to take effect.

---

#### Change 3 — UFW verification (2026-04-21 ~21:32 CEST)

**Status:** UFW is active and correctly configured — no changes needed.

**Current rules (all on tailscale0 interface only):**
- `Anywhere on tailscale0` — ALLOW IN
- `22/tcp on tailscale0` — ALLOW IN (SSH)
- `443/tcp on tailscale0` — ALLOW IN (OpenClaw Gateway)
- IPv6 equivalents of the above

**Key findings:**
- Default: deny incoming, allow outgoing
- No rules on public interfaces — all traffic must come via Tailscale
- Logging: on (low)
- No changes required

---

#### Change 4 — fail2ban hardened (2026-04-21 ~21:42-21:44 CEST)

**Problem:** fail2ban ban time was only 10 minutes with 5 retries per 10-minute window — trivial for persistent attackers to wait out and retry.

**Action:**
- Created `/etc/fail2ban/jail.d/custom-hardening.conf` — initially had no effect due to fail2ban load order (`jail.local` overrides `jail.d/*.conf`)
- Backed up `/etc/fail2ban/jail.local` to `/etc/fail2ban/jail.local.bak.2026-04-21`
- Updated `/etc/fail2ban/jail.local` directly:
  - `bantime`: 10m → 1h
  - `maxretry`: 5 → 3
  - Added progressive banning: `bantime.increment = true`, `bantime.factor = 2`, `bantime.maxtime = 1w`
  - Added SSH aggressive mode: `mode = aggressive`

**Verification:**
```
$ sudo fail2ban-client get sshd bantime
3600
$ sudo fail2ban-client get sshd maxretry
3
```

**Security Impact:** Repeat offenders now face escalating bans: 1h → 2h → 4h → 8h → up to 1 week. Aggressive mode monitors additional SSH failure types (invalid user, protocol errors, etc.).

**Key Lesson:** `fail2ban jail.local` overrides `jail.d/*.conf` — put custom settings in `jail.local`.

---

#### Change 5 — SSH daemon hardened (2026-04-21 ~21:53 CEST)

**Problem:** SSH allowed unlimited auth tries, had no idle timeout, and accepted logins from any system user.

**Action:**
- Backed up `/etc/ssh/sshd_config` to `/etc/ssh/sshd_config.bak2.2026-04-21`
- Appended hardening directives to `/etc/ssh/sshd_config`:
  - `MaxAuthTries 3` — limits per-connection auth attempts (default 6)
  - `ClientAliveInterval 300` — checks if client is alive every 5 min
  - `ClientAliveCountMax 2` — disconnects after 2 missed checks (10 min idle)
  - `AllowUsers <your-ssh-user>` — only `<your-ssh-user>` can SSH in (initially had typo `AllowUser`, caught by `sshd -t`)
- Validated config with `sudo sshd -t` before restart
- Restarted `ssh.service`

**Security Impact:** Reduces brute-force efficiency (fewer attempts per connection), drops idle sessions, and prevents any other system user (including future accidentally-created accounts) from SSH access.

**Key Lesson:** Always validate SSH config with `sshd -t` before restarting — caught `AllowUser` vs `AllowUsers` typo.

---

#### Change 6 — Unnecessary services disabled (2026-04-21 ~22:00-22:16 CEST)

**Problem:** Four services were running that serve no purpose on a headless VPS, expanding the attack surface.

**Action:**
```bash
# ModemManager — modem management
sudo systemctl disable --now ModemManager.service

# fwupd — firmware updates (DBus-activated, requires masking)
sudo systemctl mask fwupd.service
sudo systemctl stop fwupd.service

# udisks2 — disk manager (persisted after disable)
sudo systemctl stop udisks2.service
sudo systemctl mask udisks2.service

# multipathd — multipath storage (persisted via socket activation)
sudo systemctl stop multipathd.service
sudo systemctl mask multipathd.service
sudo systemctl disable --now multipathd.socket
sudo systemctl mask multipathd.socket
```

**Before:** 21 running services  
**After:** 18 running services

**Services removed:** ModemManager, fwupd, udisks2, multipathd

**Key Lessons:**
- DBus-activated services (fwupd) need `systemctl mask` instead of `disable`
- Some services (multipathd) persist via socket activation — mask the socket too

---

#### Change 7 — MOTD cleaned (2026-04-21 ~22:03 CEST)

**Problem:** Static MOTD (`/etc/motd`) displayed hosting provider branding on SSH login, aiding attacker fingerprinting.

**Action:**
```bash
sudo truncate -s 0 /etc/motd
sudo chmod -x /etc/update-motd.d/85-fwupd
sudo chmod -x /etc/update-motd.d/50-motd-news
```

---

#### Change 8 — Final verification (2026-04-21 ~22:16 CEST)

**Result:** Zero ports open on public interfaces. All listening services bound to Tailscale or localhost only.

| Address | Port | Service | Scope |
|---------|------|---------|-------|
| `<your-tailscale-ip>` | 443 | OpenClaw Gateway | Tailscale only |
| `<your-tailscale-ip>` | 22 | SSH | Tailscale only |
| `<your-tailscale-ip>` | 36084 | OpenClaw (Tailscale) | Tailscale only |
| 127.0.0.1 | 18789 | OpenClaw Gateway (local) | Localhost only |
| 127.0.0.1 | 11434 | Ollama | Localhost only |
| 127.0.0.54/53 | 53 | systemd-resolved | Localhost only |

**UFW:** Active, deny incoming by default, rules scoped to tailscale0 only.  
**fail2ban:** Active, SSH jail running, 1h ban / 3 retries / progressive banning.  
**Running services:** 18 (down from 21).

**Phase 3 Status: COMPLETE ✅**

---

## Architecture After Hardening (Phase 1)

```
Internet ──✕──► SSH (port 22)     ← BLOCKED: no public listener
Internet ──✕──► Ollama (11434)    ← BLOCKED: localhost only
Internet ──✕──► OpenClaw (18789)  ← BLOCKED: localhost only

Tailscale ──► SSH (<your-tailscale-ip>:22)       ← Only authorized path
Tailscale ──► OpenClaw GW (<your-tailscale-ip>:443)  ← Gateway API
```

---

## Remaining Work

### Phase 2: Important
- [x] Harden fail2ban (increase ban time, enable progressive banning, reduce maxretry)
- [x] Harden SSH further (MaxAuthTries, ClientAliveInterval, AllowUsers)
- [x] Verify and configure UFW rules — verified, already correct

### Phase 3: Cleanup
- [x] Disable unnecessary services (ModemManager, fwupd, udisks2, multipathd)
- [x] Remove provider branding from MOTD + disable unnecessary MOTD scripts
- [x] Final verification pass — zero ports on public interfaces, services cleaned up

### Phase 4: Future Enhancements (Not Yet Done)
- [ ] Add AppArmor or SELinux profiles for key services
- [ ] Set up AIDE or Tripwire for file integrity monitoring
- [ ] Configure auditd for detailed system call logging
- [ ] Set up automated off-site backup for critical configs
- [ ] Implement WireGuard as backup VPN (or replacement for Tailscale)
- [ ] Add log aggregation and alerting (e.g., Loki + Grafana)
- [ ] Set up certificate pinning for Tailscale

---

## Config Snapshots

All pre-change configuration files saved to: `configs/`
- `sshd_config` — main SSH daemon config
- `sshd_config.d_60-cloudimg-settings.conf` — cloud image defaults
- `sshd_config.d_tailscale-only.conf` — Tailscale SSH overrides
- `fail2ban_jail.d_defaults-debian.conf` — fail2ban jail defaults
- `fail2ban_jail.local` — fail2ban main config
- `ufw.conf` — UFW configuration
- `20auto-upgrades` — unattended upgrades config

---

## How to Use This Report

1. **Read the Initial State** — compare with your own `ss -tlnp`, `ufw status`, `systemctl list-units` output
2. **Apply changes in order** — some steps depend on previous ones (e.g., disable ssh.socket before setting ListenAddress)
3. **Validate before restarting** — always run `sshd -t` before restarting SSH
4. **Verify after each phase** — confirm with `ss -tlnp`, `ufw status verbose`, `fail2ban-client status`
5. **Keep backups** — always backup configs before editing (`cp file file.bak.YYYY-MM-DD`)

---

## License

This hardening report is provided as-is for educational and reference purposes. Review and adapt all commands for your specific environment before running them.
