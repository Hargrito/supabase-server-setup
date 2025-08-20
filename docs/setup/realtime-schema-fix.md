---
title: Realtime Schema Fix
summary: Simple fix for realtime service schema migration issues with search_path configuration
lastUpdated: 2025-08-19
owners: ["@team"]
tags: ["realtime", "postgres", "schema", "fix"]
---

# Realtime Schema Fix

## Problem
Realtime service fails with `ERROR 3F000 (invalid_schema_name) no schema has been selected to create in` because the search_path is not properly configured.

## Root Cause
- `ALTER DATABASE CURRENT_DATABASE()` syntax is invalid
- Realtime needs `_realtime` schema in search_path for migrations
- DB-level and role-level search_path not properly set

## Simple Fix

Run this script from your inventory directory:

```bash
#!/bin/bash
# File: fix-realtime-schema.sh

cd ~/supabase-docker/inventory

# Get database credentials
PW=$(grep -E '^POSTGRES_PASSWORD=' .env | cut -d= -f2)
USER=$(grep -E '^POSTGRES_USER=' .env | cut -d= -f2)
DB=$(grep -E '^POSTGRES_DB=' .env | cut -d= -f2)

echo "Fixing realtime schema configuration..."

# Fix the search_path configuration
docker run --rm --network supabase_shared -e PGPASSWORD="$PW" postgres:15 \
  psql -h shared_postgres -p 5432 -U "$USER" -d "$DB" -v ON_ERROR_STOP=1 <<SQL
-- Ensure _realtime schema exists
CREATE SCHEMA IF NOT EXISTS _realtime;

-- Set DB-level search_path (works for all users)
SELECT format('ALTER DATABASE %I SET search_path = _realtime, public', '$DB') \gexec

-- Set role-level search_path as backup
DO \$\$
BEGIN
  EXECUTE format('ALTER ROLE %I IN DATABASE %I SET search_path = _realtime, public', '$USER', '$DB');
END
\$\$;

-- Verify current search_path
SHOW search_path;
SQL

echo "Restarting realtime service..."

# Restart realtime service
docker compose rm -sf realtime
docker compose up -d realtime

# Wait and check logs
sleep 5
echo "Checking realtime logs..."
docker compose logs realtime --no-color --tail=50

# Test health endpoint
echo "Testing realtime health..."
sleep 2
curl -i http://127.0.0.1:54340/api/health || echo "Health check failed - service may still be starting"
```

## Usage

1. Make the script executable:
   ```bash
   chmod +x fix-realtime-schema.sh
   ```

2. Run the fix:
   ```bash
   ./fix-realtime-schema.sh
   ```

3. Verify realtime is working:
   ```bash
   curl -i http://127.0.0.1:54340/api/health
   ```

## Expected Output
- Realtime service should start without schema migration errors
- Health endpoint should return 200 OK
- Logs should show successful Phoenix startup

## Verification Commands

```bash
# Check if search_path is set correctly
docker run --rm --network supabase_shared -e PGPASSWORD="$PW" postgres:15 \
  psql -h shared_postgres -p 5432 -U "$USER" -d "$DB" -c "SHOW search_path;"

# Check realtime service status
docker compose ps realtime

# Check realtime logs for errors
docker compose logs realtime --tail=100 | grep -i error
```
