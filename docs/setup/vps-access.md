---
title: VPS Access & DNS Readiness
summary: Checklist and recorded details to confirm access to the Ubuntu 22.04 VPS and DNS control for subdomains.
lastUpdated: 2025-08-19
owners: ["@team"]
tags: ["vps", "access", "dns", "ssh"]
---

# VPS Access & DNS Readiness

Use this page to record connection details and confirm prerequisites before deployment.

## VPS Details
- Provider: Hostinger VPS
- OS: Ubuntu 24.04 LTS
- Public IPv4: 147.79.75.80
- Hostname: srv656044.hstgr.cloud
- Primary user: supauser (sudo)
- SSH Port: 22

## SSH Access
- [ ] Generate SSH key (if needed): `ssh-keygen -t ed25519 -C "you@example.com"`
- [ ] Add public key to `/home/supauser/.ssh/authorized_keys`
- [ ] Disable root login: `PermitRootLogin no`
- [ ] Disable password auth: `PasswordAuthentication no`
- [ ] Restart SSH: `sudo systemctl restart ssh`
- [ ] Verify login as `supauser` with key only

## DNS Readiness
- Domain registrar/provider: <to-confirm>
- Planned subdomains (alpha2web.com):
  - api-inventory.alpha2web.com
  - studio-inventory.alpha2web.com
  - api-moving.alpha2web.com
  - studio-moving.alpha2web.com
- [x] Create A records pointing to VPS IP (147.79.75.80)
- [ ] Confirm propagation:
 ```bash
 dig +short api-inventory.alpha2web.com
 dig +short studio-inventory.alpha2web.com
 dig +short api-moving.alpha2web.com
 dig +short studio-moving.alpha2web.com
 ```

## Notes
- Keep fingerprints and last test time here for audit.
- Consider changing SSH port only after Fail2Ban in place.
