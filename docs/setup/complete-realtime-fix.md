---
title: Complete Realtime Fix
summary: Complete solution for realtime service including schema creation, migrations, and port mapping
lastUpdated: 2025-08-19
owners: ["@team"]
tags: ["realtime", "postgres", "schema", "complete-fix"]
---

# Complete Realtime Fix

## Root Cause
1. `_realtime` schema doesn't exist in the database
2. Port mapping missing from docker-compose.yml
3. Schema migrations table not created

## Complete Fix Script

```bash
cd ~/supabase-docker/inventory

# Get credentials
PW=$(grep POSTGRES_PASSWORD .env | cut -d= -f2)
USER=$(grep POSTGRES_USER .env | cut -d= -f2)
DB=$(grep POSTGRES_DB .env | cut -d= -f2)

# Step 1: Create _realtime schema and all required setup
docker run --rm --network supabase_shared -e PGPASSWORD="$PW" postgres:15 \
  psql -h shared_postgres -p 5432 -U "$USER" -d "$DB" -v ON_ERROR_STOP=1 <<SQL
-- Create _realtime schema
CREATE SCHEMA IF NOT EXISTS _realtime AUTHORIZATION $USER;

-- Set search_path at database level
ALTER DATABASE inventory_db SET search_path = _realtime, public;

-- Set search_path at role level
ALTER ROLE $USER IN DATABASE inventory_db SET search_path = _realtime, public;

-- Create schema_migrations table in _realtime schema
CREATE TABLE IF NOT EXISTS _realtime.schema_migrations (
  version bigint NOT NULL PRIMARY KEY,
  inserted_at timestamp(0) without time zone DEFAULT NOW()
);

-- Create basic realtime tables to prevent migration errors
CREATE TABLE IF NOT EXISTS _realtime.tenants (
  id uuid NOT NULL PRIMARY KEY DEFAULT gen_random_uuid(),
  name text NOT NULL,
  external_id text,
  jwt_secret text,
  max_concurrent_users integer DEFAULT 200,
  inserted_at timestamp(0) without time zone DEFAULT NOW(),
  updated_at timestamp(0) without time zone DEFAULT NOW()
);

-- Insert default tenant
INSERT INTO _realtime.tenants (name, external_id, jwt_secret) 
VALUES ('realtime', 'realtime', 'your-jwt-secret-here')
ON CONFLICT (id) DO NOTHING;

-- Grant permissions
GRANT ALL ON SCHEMA _realtime TO $USER;
GRANT ALL ON ALL TABLES IN SCHEMA _realtime TO $USER;
GRANT ALL ON ALL SEQUENCES IN SCHEMA _realtime TO $USER;

-- Verify setup
\dn
SHOW search_path;
SQL

# Step 2: Ensure port mapping in docker-compose.yml
if ! grep -q "54340:4000" docker-compose.yml; then
  echo "Adding port mapping to realtime service..."
  sed -i '/realtime:/,/restart:/ s/restart:/ports:\n      - "127.0.0.1:54340:4000"\n    restart:/' docker-compose.yml
fi

# Step 3: Restart realtime service
docker compose rm -sf realtime
docker compose up -d realtime

# Step 4: Wait and check logs
sleep 10
echo "=== Realtime Logs ==="
docker compose logs realtime --tail=30

# Step 5: Verify service status
echo "=== Service Status ==="
docker compose ps realtime

# Step 6: Test health endpoint
echo "=== Health Check ==="
for i in {1..5}; do
  echo "Attempt $i:"
  curl -s -o /dev/null -w "HTTP Status: %{http_code}\n" http://127.0.0.1:54340/api/health
  sleep 2
done
```

## Expected Output
- Schema creation should succeed
- Realtime service should show "Up" status
- Health endpoint should return 200

## If Still Failing
Check the specific error in logs:
```bash
docker compose logs realtime --tail=50 | grep -i error
```
