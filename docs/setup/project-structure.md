---
title: Project Structure & Conventions
summary: Directory layout and conventions for multi-stack Supabase deployment with shared Postgres.
lastUpdated: 2025-08-19
owners: ["@team"]
tags: ["structure", "conventions", "docs"]
---

# Project Structure & Conventions

## Host filesystem layout
```
~/supabase-docker/
  shared/           # shared Postgres compose + init scripts
  backups/          # backup scripts and artifacts
  inventory/        # inventory stack files
  moving/           # moving stack files
  <newstack>/       # future stacks
```

## Repo docs layout
```
docs/
  setup/
    vps-access.md
    docker-install.md
    project-structure.md
    shared-postgres.md
    add-a-stack.md
  templates/
    stack-template/
      README.md
      .env.example
      docker-compose.yml.example
      kong.yml.example
```

## Conventions
- Use `.env` per stack with all ports and secrets.
- Bind app services to `127.0.0.1` only; expose via Nginx.
- Follow port allocation strategy: base + offsets per stack.
- Keep secrets out of repo; use encrypted secrets system in apps.
- Keep runbooks small and focused; link from the main plan.
