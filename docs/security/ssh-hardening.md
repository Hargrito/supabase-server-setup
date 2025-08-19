---
title: SSH Hardening (Ubuntu 22.04/24.04)
summary: Secure SSH by using key-based auth with a non-root sudo user and disabling password and root logins.
lastUpdated: 2025-08-19
owners: ["@team"]
tags: ["security", "ssh", "hardening"]
---

# SSH Hardening (Ubuntu 22.04/24.04)

## Create a non-root sudo user
```bash
sudo adduser supauser
sudo usermod -aG sudo supauser
```

## Add SSH key for the user
```bash
sudo -u supauser mkdir -p /home/supauser/.ssh
sudo -u supauser chmod 700 /home/supauser/.ssh
# Paste your public key:
echo "ssh-ed25519 AAAA... your@email" | sudo tee /home/supauser/.ssh/authorized_keys
sudo chmod 600 /home/supauser/.ssh/authorized_keys
sudo chown -R supauser:supauser /home/supauser/.ssh
```

## SSH daemon configuration
Edit `/etc/ssh/sshd_config`:
```
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
```
Reload SSH:
```bash
sudo systemctl restart ssh
```

## Record server host key fingerprints (recommended)
On the server, list and note fingerprints to verify on first client connect:
```bash
sudo ssh-keygen -l -f /etc/ssh/ssh_host_ed25519_key.pub
sudo ssh-keygen -l -f /etc/ssh/ssh_host_rsa_key.pub
```

## Resolve host key change warnings on client
If you reinstalled/reimaged the server or changed SSH keys, clients may see:
`WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!`

On your workstation, remove the old entry and accept the new key:
```powershell
ssh-keygen -R <server_ip>
ssh -o StrictHostKeyChecking=accept-new supauser@<server_ip>
```
Verify the new fingerprint matches what you recorded from the server before proceeding.
```

## Verify
From your workstation:
```bash
ssh -o PreferredAuthentications=publickey supauser@<server_ip>
```
If successful, record fingerprints and the test date below.

## Record
- Fingerprint: <fill>
- Tested by: <name>
- Date: <YYYY-MM-DD>
