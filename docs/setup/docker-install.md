---
title: Install Docker & Compose on Ubuntu 22.04
summary: Steps to install Docker Engine and the Docker Compose plugin on Ubuntu 22.04 LTS and verify the setup.
lastUpdated: 2025-08-19
owners: ["@team"]
tags: ["docker", "compose", "ubuntu22"]
---

# Install Docker & Compose on Ubuntu 22.04

## Steps

1) Remove old packages (if present)
```bash
sudo apt remove -y docker docker-engine docker.io containerd runc || true
```

2) Install prerequisites
```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release
```

3) Add Docker’s official GPG key and repo
```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

4) Install Docker Engine, CLI, containerd, and Compose plugin
```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

5) Enable and test Docker
```bash
sudo systemctl enable --now docker
sudo docker run --rm hello-world
```

6) Use Docker without sudo
```bash
sudo usermod -aG docker $USER
newgrp docker
```

7) Verify versions
```bash
docker --version
docker compose version
```

## Notes
- For convenience-only installs, you may use the `get.docker.com` script as in the main plan.
- Ensure your user (e.g., `supauser`) is in the `docker` group.
