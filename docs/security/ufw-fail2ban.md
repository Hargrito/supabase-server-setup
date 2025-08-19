---
title: UFW Firewall & Fail2Ban
summary: Configure UFW to allow SSH/HTTP/HTTPS and enable Fail2Ban to protect SSH on Ubuntu 22.04.
lastUpdated: 2025-08-19
owners: ["@team"]
tags: ["security", "ufw", "fail2ban"]
---

# UFW Firewall & Fail2Ban

## UFW configuration
```bash
sudo apt update && sudo apt install -y ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH   # 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw --force enable
sudo ufw status verbose
```

## Fail2Ban installation and basic SSH jail
```bash
sudo apt install -y fail2ban
sudo systemctl enable --now fail2ban
```

Create `/etc/fail2ban/jail.local` (optional hardening):
```
[sshd]
enabled = true
port    = ssh
logpath = /var/log/auth.log
backend = systemd
maxretry = 5
bantime = 1h
findtime = 10m
```
Apply changes:
```bash
sudo systemctl restart fail2ban
sudo fail2ban-client status sshd
```

## Notes
- If you later change SSH port, update both UFW rule and `[sshd]` jail port.
- Consider jails for Nginx auth if exposing admin paths.
