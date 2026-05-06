# VPS Security Hardening Report

> Complete Ubuntu 24.04 VPS hardening guide — from exposed SSH to zero public attack surface.

This repository documents a real-world security hardening of a Ubuntu 24.04 VPS hosting an AI assistant platform (OpenClaw + Ollama). The server was initially exposed with SSH open to the entire internet and weak brute-force protection. This guide walks through every change made to lock it down.

---

## What Was Done

### Phase 1: Critical Fixes ✅
1. **SSH restricted to Tailscale only** — Removed public SSH access, requires Tailscale VPN
2. **Disabled systemd ssh.socket** — Fixed Ubuntu 24.04 socket activation overriding sshd_config
3. **UFW verified** — Confirmed deny-by-default, Tailscale-scoped rules
4. **fail2ban hardened** — 1h ban, 3 retries, progressive banning up to 1 week, aggressive mode
5. **SSH daemon locked down** — MaxAuthTries 3, 10min idle timeout, AllowUsers restriction
6. **Unnecessary services removed** — ModemManager, fwupd, udisks2, multipathd (21→18 services)
7. **MOTD cleaned** — Removed provider branding, disabled news/ad scripts
8. **Final verification** — Zero ports on public interfaces

### Phase 2: Cleanup ✅
- All Phase 1 items completed and verified

### Phase 3: Future Enhancements 🔄
- [ ] AppArmor/SELinux profiles
- [ ] AIDE/Tripwire file integrity monitoring
- [ ] auditd system call logging
- [ ] Automated off-site config backups
- [ ] WireGuard backup VPN
- [ ] Log aggregation + alerting (Loki + Grafana)
- [ ] Certificate pinning for Tailscale

---

## Key Lessons Learned

| Lesson | Context |
|--------|---------|
| **Ubuntu 24.04 ssh.socket** | `ssh.socket` overrides `sshd_config` ListenAddress — must disable it |
| **fail2ban jail.local** | `jail.local` overrides `jail.d/*.conf` — custom settings go in `jail.local` |
| **Always validate SSH** | `sshd -t` catches typos like `AllowUser` vs `AllowUsers` |
| **DBus services** | `fwupd` needs `systemctl mask`, not just `disable` |
| **Socket activation** | `multipathd` persists via socket — mask the socket too |

---

## Files

| File | Description |
|------|-------------|
| `hardening-report.md` | Full narrative report with all changes, timestamps, and verification |
| `configs/` | Sanitized configuration backups (sshd, fail2ban, UFW, auto-upgrades) |

---

## How to Use This Guide

1. **Start with the Initial State** — run `ss -tlnp`, `ufw status`, `systemctl list-units` on your server
2. **Apply changes in order** — some steps depend on previous ones
3. **Validate before restarting** — `sshd -t` before any SSH restart
4. **Verify after each phase** — `ss -tlnp`, `ufw status verbose`, `fail2ban-client status`
5. **Always backup** — `cp file file.bak.YYYY-MM-DD`

---

## Architecture After Hardening

```
Internet ──✕──► SSH (port 22)     ← BLOCKED
Internet ──✕──► Ollama (11434)    ← BLOCKED
Internet ──✕──► OpenClaw (18789)  ← BLOCKED

Tailscale VPN ──► SSH (22)         ← Only authorized path
Tailscale VPN ──► OpenClaw (443)   ← Gateway API
```

**Result:** Zero listening ports on public interfaces. All access requires Tailscale VPN.

---

## Requirements

- Ubuntu 24.04 LTS (Noble Numbat)
- Tailscale installed and configured
- Root or sudo access
- `ss`, `ufw`, `fail2ban`, `systemctl` available

---

## License

This report is provided as-is for educational and reference purposes. Review and adapt all commands for your specific environment before running them.

---

## Contributing

This is a living document. Future hardening phases will be added as commits with dates. See `hardening-report.md` for the full changelog.
