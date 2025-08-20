---
title: Phase Readiness Checklist
summary: Verification checklist to confirm completion of phases 0-5 before proceeding to Phase 6
lastUpdated: 2025-08-19
owners: ["@team"]
tags: ["checklist", "deployment", "verification"]
---

# Phase Readiness Checklist

Run these commands to verify each phase is complete before proceeding to Phase 6.

## Phase 0: VPS Access & DNS Readiness
```bash
# Verify SSH access works
ssh supauser@<server_ip> "whoami && pwd"

# Check UFW status (should show 22/tcp ALLOW)
ssh supauser@<server_ip> "sudo ufw status"
```
**Expected**: SSH works, UFW shows port 22 allowed

## Phase 1: Base System Hardening
```bash
# Check SSH config
ssh supauser@<server_ip> "sudo grep -E '^(PermitRootLogin|PasswordAuthentication)' /etc/ssh/sshd_config"

# Check Fail2Ban status
ssh supauser@<server_ip> "sudo systemctl is-active fail2ban"

# Check UFW rules
ssh supauser@<server_ip> "sudo ufw status numbered"
```
**Expected**: 
- `PermitRootLogin no`
- `PasswordAuthentication no` 
- Fail2Ban active
- UFW shows 22, 80, 443 allowed

## Phase 2: Docker & Project Scaffold
```bash
# Check Docker installation
ssh supauser@<server_ip> "docker --version && docker compose version"

# Check project structure
ssh supauser@<server_ip> "ls -la ~/supabase-docker/"
```
**Expected**: Docker installed, directories exist: `shared/`, `inventory/`, `moving/`

## Phase 3: Shared Postgres
```bash
# Check shared postgres is running
ssh supauser@<server_ip> "cd ~/supabase-docker/shared && docker compose ps"

# Check databases exist
ssh supauser@<server_ip> "cd ~/supabase-docker/shared && docker exec -i \$(docker compose ps -q postgres) psql -U supabase -d postgres -c '\l' | grep -E '(inventory_db|moving_db)'"

# Check network exists
ssh supauser@<server_ip> "docker network ls | grep supabase_shared"
```
**Expected**: 
- Postgres container running
- `inventory_db` and `moving_db` exist
- `supabase_shared` network exists

## Phase 4: Inventory Scaffold
```bash
# Check inventory directory and .env
ssh supauser@<server_ip> "ls -la ~/supabase-docker/inventory/"
ssh supauser@<server_ip> "cd ~/supabase-docker/inventory && grep -E '^(POSTGRES_|PROJECT_NAME)' .env"
```
**Expected**: `.env` file exists with correct database settings

## Phase 5: Inventory Stack Running
```bash
# Check all schemas and fixes applied
ssh supauser@<server_ip> "cd ~/supabase-docker/inventory && PW=\$(grep POSTGRES_PASSWORD .env | cut -d= -f2) && docker exec -i shared-postgres-1 psql -U supabase -d inventory_db -c '\dn' | grep -E '(auth|_realtime)'"

# Check search_path configuration
ssh supauser@<server_ip> "cd ~/supabase-docker/inventory && PW=\$(grep POSTGRES_PASSWORD .env | cut -d= -f2) && docker exec -i shared-postgres-1 psql -U supabase -d inventory_db -c 'SHOW search_path'"

# Check inventory services running
ssh supauser@<server_ip> "cd ~/supabase-docker/inventory && docker compose ps"

# Test service endpoints
ssh supauser@<server_ip> "curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:5555/health"
ssh supauser@<server_ip> "curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:5512"
ssh supauser@<server_ip> "curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:54340/api/health"
```
**Expected**:
- `auth` and `_realtime` schemas exist
- search_path shows `_realtime, public`
- All services show "Up" status
- All endpoints return 200

## Ready for Phase 6?
If all checks pass, you're ready to proceed with Phase 6: Moving Stack.

## Quick Fix Commands
If any checks fail, here are the key fixes:

### Apply realtime schema fix:
```bash
cd ~/supabase-docker/inventory
PW=$(grep -E '^POSTGRES_PASSWORD=' .env | cut -d= -f2)
USER=$(grep -E '^POSTGRES_USER=' .env | cut -d= -f2)
DB=$(grep -E '^POSTGRES_DB=' .env | cut -d= -f2)

docker run --rm --network supabase_shared -e PGPASSWORD="$PW" postgres:15 \
  psql -h shared_postgres -p 5432 -U "$USER" -d "$DB" -v ON_ERROR_STOP=1 <<SQL
CREATE SCHEMA IF NOT EXISTS _realtime;
SELECT format('ALTER DATABASE %I SET search_path = _realtime, public', '$DB') \gexec
DO \$\$
BEGIN
  EXECUTE format('ALTER ROLE %I IN DATABASE %I SET search_path = _realtime, public', '$USER', '$DB');
END
\$\$;
SQL

docker compose rm -sf realtime
docker compose up -d realtime
```
