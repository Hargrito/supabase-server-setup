---
title: Quick Deployment Guide
summary: Simplified step-by-step guide to get Supabase working with realtime service
lastUpdated: 2025-08-19
owners: ["@team"]
tags: ["deployment", "quick-start", "realtime", "fix"]
---

# Quick Deployment Guide

This is the simplified, working deployment sequence based on the documented fixes.

## Prerequisites
- VPS with Ubuntu 24.04
- SSH access as `supauser`
- Docker and Docker Compose installed

## Step 1: Create Network and Shared Postgres

```bash
# Create Docker network
docker network create supabase_shared

# Create shared postgres directory
mkdir -p ~/supabase-docker/shared
cd ~/supabase-docker/shared

# Create .env
cat > .env << 'EOF'
POSTGRES_USER=supabase
POSTGRES_PASSWORD=your_strong_password_here
POSTGRES_DB=postgres
TIMEZONE=UTC
EOF

# Create docker-compose.yml
cat > docker-compose.yml << 'EOF'
services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init-databases.sql:/docker-entrypoint-initdb.d/init-databases.sql
    ports:
      - "127.0.0.1:5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped
    networks:
      supabase_shared:
        aliases:
          - shared_postgres

volumes:
  postgres_data:

networks:
  supabase_shared:
    external: true
EOF

# Create init script
cat > init-databases.sql << 'EOF'
CREATE DATABASE inventory_db;
CREATE DATABASE moving_db;
EOF

# Start postgres
docker compose up -d
```

## Step 2: Create Inventory Stack

```bash
mkdir -p ~/supabase-docker/inventory
cd ~/supabase-docker/inventory

# Create .env (use same password as shared postgres)
cat > .env << 'EOF'
PROJECT_NAME=inventory
TIMEZONE=UTC
POSTGRES_HOST=shared_postgres
POSTGRES_PORT=5432
POSTGRES_USER=supabase
POSTGRES_PASSWORD=your_strong_password_here
POSTGRES_DB=inventory_db
DATABASE_URL=postgres://${POSTGRES_USER}:${POSTGRES_PASSWORD}@${POSTGRES_HOST}:${POSTGRES_PORT}/${POSTGRES_DB}

# Generate these with: openssl rand -base64 32
JWT_SECRET=your_jwt_secret_here
ANON_KEY=your_anon_key_here
SERVICE_ROLE_KEY=your_service_role_key_here

# API URLs (local for now)
API_EXTERNAL_URL=http://127.0.0.1:5555
SUPABASE_PUBLIC_URL=http://127.0.0.1:5555

# Ports
AUTH_PORT=5555
REST_PORT=5512
REALTIME_PORT=54340
STUDIO_PORT=3001
EOF
```

## Step 3: Apply Database Fixes

```bash
cd ~/supabase-docker/inventory

# Get credentials
PW=$(grep -E '^POSTGRES_PASSWORD=' .env | cut -d= -f2)
USER=$(grep -E '^POSTGRES_USER=' .env | cut -d= -f2)
DB=$(grep -E '^POSTGRES_DB=' .env | cut -d= -f2)

# Apply all required fixes
docker run --rm --network supabase_shared -e PGPASSWORD="$PW" postgres:15 \
  psql -h shared_postgres -p 5432 -U "$USER" -d "$DB" -v ON_ERROR_STOP=1 <<SQL
-- Create auth schema
CREATE SCHEMA IF NOT EXISTS auth AUTHORIZATION $USER;

-- Create postgres role for migrations
DO \$\$ BEGIN 
  IF NOT EXISTS (SELECT 1 FROM pg_roles WHERE rolname = 'postgres') THEN 
    CREATE ROLE postgres; 
  END IF; 
END \$\$;

-- Create auth.factor_type enum
DO \$\$
BEGIN
  IF NOT EXISTS (
    SELECT 1 FROM pg_type t JOIN pg_namespace n ON n.oid = t.typnamespace
    WHERE t.typname = 'factor_type' AND n.nspname = 'auth'
  ) THEN
    CREATE TYPE auth.factor_type AS ENUM ('totp','webauthn');
  END IF;
END \$\$;

-- Create _realtime schema
CREATE SCHEMA IF NOT EXISTS _realtime;

-- Set DB-level search_path (CRITICAL for realtime)
SELECT format('ALTER DATABASE %I SET search_path = _realtime, public', '$DB') \gexec

-- Set role-level search_path as backup
DO \$\$
BEGIN
  EXECUTE format('ALTER ROLE %I IN DATABASE %I SET search_path = _realtime, public', '$USER', '$DB');
END
\$\$;

-- Verify search_path
SHOW search_path;
SQL
```

## Step 4: Create Minimal Docker Compose

```bash
cd ~/supabase-docker/inventory

cat > docker-compose.yml << 'EOF'
services:
  auth:
    image: supabase/gotrue:v2.99.0
    environment:
      GOTRUE_API_HOST: 0.0.0.0
      GOTRUE_API_PORT: 9999
      GOTRUE_DB_DRIVER: postgres
      GOTRUE_DB_NAMESPACE: auth
      GOTRUE_DB_DATABASE_URL: postgres://${POSTGRES_USER}:${POSTGRES_PASSWORD}@${POSTGRES_HOST}:${POSTGRES_PORT}/${POSTGRES_DB}?search_path=auth
      GOTRUE_SITE_URL: ${SUPABASE_PUBLIC_URL}
      GOTRUE_URI_ALLOW_LIST: ${SUPABASE_PUBLIC_URL}
      GOTRUE_JWT_SECRET: ${JWT_SECRET}
      GOTRUE_JWT_EXP: 3600
      GOTRUE_JWT_DEFAULT_GROUP_NAME: authenticated
      GOTRUE_JWT_ADMIN_ROLES: service_role
      GOTRUE_JWT_AUD: authenticated
      API_EXTERNAL_URL: ${API_EXTERNAL_URL}
    ports:
      - "127.0.0.1:${AUTH_PORT}:9999"
    restart: unless-stopped
    networks:
      - supabase_shared

  rest:
    image: postgrest/postgrest:v11.2.0
    environment:
      PGRST_DB_URI: postgres://${POSTGRES_USER}:${POSTGRES_PASSWORD}@${POSTGRES_HOST}:${POSTGRES_PORT}/${POSTGRES_DB}
      PGRST_DB_SCHEMAS: public
      PGRST_DB_ANON_ROLE: anon
      PGRST_JWT_SECRET: ${JWT_SECRET}
      PGRST_DB_USE_LEGACY_GUCS: "false"
    ports:
      - "127.0.0.1:${REST_PORT}:3000"
    restart: unless-stopped
    networks:
      - supabase_shared

  realtime:
    image: supabase/realtime:v2.25.35
    environment:
      PORT: 4000
      DB_HOST: ${POSTGRES_HOST}
      DB_PORT: ${POSTGRES_PORT}
      DB_USER: ${POSTGRES_USER}
      DB_PASSWORD: ${POSTGRES_PASSWORD}
      DB_NAME: ${POSTGRES_DB}
      DB_AFTER_CONNECT_QUERY: 'SET search_path TO _realtime'
      DB_ENC_KEY: supabaserealtime
      API_JWT_SECRET: ${JWT_SECRET}
      FLY_ALLOC_ID: fly123
      FLY_APP_NAME: realtime
      SECRET_KEY_BASE: UpNVntn3cDxHJpq99YMc1T1AQgQpc8kfYTuRgBiYa15BLrx8etQoXz3gZv1/u2oq
      ERL_AFLAGS: -proto_dist inet_tcp
      ENABLE_TAILSCALE: "false"
      DNS_NODES: "''"
    ports:
      - "127.0.0.1:${REALTIME_PORT}:4000"
    command: >
      sh -c "/app/bin/migrate && /app/bin/realtime eval 'Realtime.Release.seeds(Realtime.Repo)' && /app/bin/server"
    restart: unless-stopped
    networks:
      - supabase_shared

networks:
  supabase_shared:
    external: true
EOF
```

## Step 5: Start Services

```bash
cd ~/supabase-docker/inventory

# Start auth service first
docker compose up -d auth
sleep 3
docker compose logs auth --tail=50

# Start rest service
docker compose up -d rest
sleep 3
docker compose logs rest --tail=50

# Start realtime service
docker compose up -d realtime
sleep 5
docker compose logs realtime --tail=50
```

## Step 6: Verify Everything Works

```bash
# Check all services are running
docker compose ps

# Test auth service
curl -i http://127.0.0.1:5555/health

# Test rest service
curl -i http://127.0.0.1:5512

# Test realtime service
curl -i http://127.0.0.1:54340/api/health
```

## Expected Results
- All services should show as "Up" in `docker compose ps`
- Auth health endpoint returns 200
- REST endpoint returns OpenAPI JSON
- Realtime health endpoint returns 200
- No schema migration errors in logs

## Troubleshooting
If realtime still fails:
1. Check search_path: `docker exec -it shared_postgres-postgres-1 psql -U supabase -d inventory_db -c "SHOW search_path;"`
2. Verify _realtime schema exists: `docker exec -it shared_postgres-postgres-1 psql -U supabase -d inventory_db -c "\dn"`
3. Check realtime logs: `docker compose logs realtime --tail=100`

## Next Steps
Once this basic setup works:
1. Add Kong gateway for API routing
2. Add Studio for database management
3. Configure Nginx with TLS certificates
4. Set up proper domain names
