---
title: Add a New Supabase Stack
summary: Procedure and checklist to add an additional Supabase project stack on the shared VPS.
lastUpdated: 2025-08-19
owners: ["@team"]
tags: ["supabase", "stack", "template", "nginx", "tls"]
---

# Add a New Supabase Stack

This guide describes how to add a new Supabase stack (API, Studio, services) to the existing VPS that hosts multiple stacks against a shared Postgres instance.

## Prerequisites
- Shared Postgres running (`~/supabase-docker/shared/`).
- Docker network `supabase_shared` exists.
- DNS control for `api-<stack>.yourdomain.com` and `studio-<stack>.yourdomain.com`.
- Unique secrets (JWT, anon, service role) generated per stack.

## Port Allocation Strategy
To avoid conflicts, allocate ports deterministically:
- Kong HTTP: 8000 + 100*(N-1)
- Kong HTTPS: 8443 + 100*(N-1)
- Studio: 3000 + N
- PgBouncer: 6431 + N
Where N is the stack index (Inventory=1, Moving=2, new stack=3, ...).

Record chosen ports in the stack `.env` file.

## Steps

1) Create database and user (optional)
```sql
-- In shared Postgres (via psql)
CREATE DATABASE <stack>_db;
CREATE USER <stack>_user WITH PASSWORD '<strong_password>';
GRANT ALL PRIVILEGES ON DATABASE <stack>_db TO <stack>_user;
```

2) Create stack directory
```bash
mkdir -p ~/supabase-docker/<stack>
cd ~/supabase-docker/<stack>
```
Copy the template files from `docs/templates/stack-template/`:
- `.env.example` → `.env` (fill values)
- `docker-compose.yml.example` → `docker-compose.yml`
- `kong.yml.example` → `kong.yml`
- Create `functions/` as needed

3) Fill `.env`
```env
PROJECT_NAME=<stack>
POSTGRES_HOST=postgres
POSTGRES_PORT=5432
POSTGRES_DB=<stack>_db
POSTGRES_USER=supabase
POSTGRES_PASSWORD=<shared_db_password>

JWT_SECRET=<generated>
ANON_KEY=<generated>
SERVICE_ROLE_KEY=<generated>

API_EXTERNAL_URL=https://api-<stack>.yourdomain.com
SUPABASE_PUBLIC_URL=https://api-<stack>.yourdomain.com

STUDIO_PORT=<allocated>
KONG_HTTP_PORT=<allocated>
KONG_HTTPS_PORT=<allocated>
PGBOUNCER_PORT=<allocated>
```

4) Start the stack
```bash
cd ~/supabase-docker/<stack>
docker compose up -d
```
Verify: `docker ps` shows all services healthy.

5) Nginx + TLS
- Create sites for `api-<stack>` proxy to `127.0.0.1:<KONG_HTTP_PORT>`
- Create site for `studio-<stack>` proxy to `127.0.0.1:<STUDIO_PORT>`
- Enable sites, reload Nginx, obtain certs via Certbot

6) Backups & Monitoring
- Ensure DB is included in backup script.
- Add endpoints to uptime monitoring.
- Optionally add postgres exporter labels for per-db metrics.

7) Secrets integration
- Add new stack keys/URLs to encrypted secrets system and application configs.

## Checklist
- [ ] Database created and permissions set
- [ ] `.env` completed with unique secrets and allocated ports
- [ ] `docker compose up -d` runs without errors
- [ ] Nginx sites enabled; TLS certificates issued
- [ ] Backups and monitoring updated
- [ ] Secrets integrated in applications

## Troubleshooting
- Port in use: adjust allocation following strategy above.
- 502 Gateway: check container health and Nginx upstreams.
- DB connect issues: verify DB name/user/password; container can reach `postgres` host in the Docker network.
