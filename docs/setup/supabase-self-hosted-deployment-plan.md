---
title: Supabase Self-Hosted Deployment Plan (Dedicated VPS)
summary: Comprehensive guide for deploying and operating two Supabase stacks on a single dedicated VPS with shared Postgres, TLS, backups, and monitoring.
lastUpdated: 2025-08-19
owners: ["@team"]
tags: ["supabase", "postgres", "docker", "nginx", "security", "backups"]
---

> Note: This document may contain out-of-date values (e.g., references to Ubuntu 22.04 vs. current 24.04). Review and update before running in production.

# Supabase Self-Hosted Deployment Plan (Dedicated VPS)

This document provides a comprehensive deployment guide for self-hosting Supabase on a dedicated VPS, following security best practices.

## Related Docs

- Add a Stack guide: `docs/setup/add-a-stack.md` (optional)
- Stack template folder: `docs/templates/stack-template/` (optional; not required for minimal scaffolds)

---

## 1) Environment Overview

- **Provider**: Hostinger VPS (dedicated for Supabase)
- **OS**: Ubuntu 24.04 LTS
- **Specs**: 4 vCPU, 16 GB RAM, 200 GB NVMe, 16 TB bandwidth
- **Purpose**: Multi-project Supabase hosting (Inventory Builder + Moving/AI Agent)
- **Architecture**: Two isolated Supabase stacks on one VPS
- **Domains**: 
  - api-inventory.yourdomain.com, studio-inventory.yourdomain.com
  - api-moving.yourdomain.com, studio-moving.yourdomain.com

---

## 2) High-Level Milestones

- [ ] Provision VPS and base hardening
- [ ] Docker + Compose install and project scaffold
- [ ] Shared Postgres setup with two databases
- [ ] Deploy two Supabase stacks (Inventory + Moving projects)
- [ ] Domain + HTTPS via Let's Encrypt (Nginx)
- [ ] Configure unique keys per stack (JWT, anon, service role)
- [ ] Set up PgBouncer per stack for connection pooling
- [ ] Implement backup strategy (nightly dumps + weekly base backup)
- [ ] Configure monitoring and alerting
- [ ] Integration with existing encrypted secrets system

### Milestones Tracker

- [ ] Phase 0: Confirm VPS access & DNS readiness
- [ ] Phase 1: Base system hardening (SSH, UFW, Fail2Ban)
- [ ] Phase 2: Docker & project scaffold
- [ ] Phase 3: Shared Postgres provisioned
- [ ] Phase 4: Minimal Inventory & Moving scaffolds prepared (no template required)
- [ ] Phase 5: Inventory stack configured and running
- [ ] Phase 6: Moving stack configured and running
- [ ] Phase 7: Nginx sites + TLS certificates
- [ ] Phase 8: Backups scheduled and tested restore
- [ ] Phase 9: Monitoring (system, Postgres, uptime)
- [ ] Phase 10: Encrypted secrets integrated
- [ ] Phase 11: App-level security hardening complete
- [ ] Phase 12: Verification & handover checklist complete

---

## Phase Runbook and Key Fixes (0–5)

This records the exact steps and conditions that made the stack work during initial bring-up.

### Phase 0: Access & DNS readiness
- Verify SSH access for `supauser`. Confirm UFW allows 22 only initially.
- Prepare DNS records (A/AAAA) but keep services bound to 127.0.0.1 until Nginx/Kong.

### Phase 1: Base hardening
- Apply SSH hardening and enable UFW + Fail2Ban as per `## 3) Server Preparation Checklist`.

### Phase 2: Docker & scaffold
- Install Docker/Compose and create `~/supabase-docker/{shared,inventory,moving}` as per `## 4) Docker & Compose Setup` and `## 4–5 minimal scaffolds`.

### Phase 3: Shared Postgres
- Bring up shared Postgres on 127.0.0.1:5432 and create `inventory_db` + `moving_db`.
- Network alias used: `shared_postgres` on `supabase_shared` network.

### Phase 4: Inventory scaffold
- Created minimal Inventory project directory and `.env` with DB connectivity.

### Phase 5: Inventory stack bring-up (Auth + REST + Realtime)
- Ensure API loopback binding and external URL placeholder:
  ```bash
  cd ~/supabase-docker/inventory
  grep -q '^API_EXTERNAL_URL=' .env || echo 'API_EXTERNAL_URL=http://127.0.0.1:5555' >> .env
  sed -i 's|^API_EXTERNAL_URL=.*|API_EXTERNAL_URL=http://127.0.0.1:5555|' .env
  grep ^API_EXTERNAL_URL= .env
  ```
- Inject `GOTRUE_DB_NAMESPACE` to `auth` service (so GoTrue uses `auth` schema):
  ```bash
  sed -i '/GOTRUE_DB_DRIVER: "postgres"/a\      GOTRUE_DB_NAMESPACE: auth' docker-compose.yml
  ```
- Fix 1: Create `auth` schema before GoTrue migrations (ownership: `${POSTGRES_USER}`):
  ```bash
  PW=$(grep -E '^POSTGRES_PASSWORD=' .env | cut -d= -f2)
  USER=$(grep -E '^POSTGRES_USER=' .env | cut -d= -f2)
  DB=$(grep -E '^POSTGRES_DB=' .env | cut -d= -f2)
  docker run --rm --network supabase_shared -e PGPASSWORD="$PW" postgres:15 \
    psql -h shared_postgres -p 5432 -U "$USER" -d "$DB" -v ON_ERROR_STOP=1 -c "
  CREATE SCHEMA IF NOT EXISTS auth AUTHORIZATION $USER;
  "
  ```
- Fix 2: Satisfy migration grants to non-existent role `postgres` by creating a NOLOGIN role:
  ```bash
  docker run --rm --network supabase_shared -e PGPASSWORD="$PW" postgres:15 \
    psql -h shared_postgres -p 5432 -U "$USER" -d "$DB" -v ON_ERROR_STOP=1 -c "
  DO $$ BEGIN IF NOT EXISTS (SELECT 1 FROM pg_roles WHERE rolname = 'postgres') THEN CREATE ROLE postgres; END IF; END $$;
  "
  ```
- Fix 3: Create missing enum `auth.factor_type` so later migrations can `ADD VALUE 'phone'`:
  ```bash
  docker run --rm --network supabase_shared -e PGPASSWORD="$PW" postgres:15 \
    psql -h shared_postgres -p 5432 -U "$USER" -d "$DB" -v ON_ERROR_STOP=1 -c "
  DO $$
  BEGIN
    IF NOT EXISTS (
      SELECT 1 FROM pg_type t JOIN pg_namespace n ON n.oid = t.typnamespace
      WHERE t.typname = 'factor_type' AND n.nspname = 'auth'
    ) THEN
      CREATE TYPE auth.factor_type AS ENUM ('totp','webauthn');
    END IF;
  END $$;
  "
  ```
- Fix 4: Configure realtime schema and search_path (CRITICAL FIX):
  ```bash
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
  
  -- Verify search_path
  SHOW search_path;
  SQL
  ```
- Recreate Auth and verify successful migrations and port binding:
  ```bash
  docker compose up -d --force-recreate auth
  docker compose logs auth --no-color --tail=200
  ss -ltnp | grep 5555   # Expect docker-proxy listening on 127.0.0.1:5555
  ```
- Start Realtime service and verify:
  ```bash
  docker compose up -d realtime
  sleep 5
  docker compose logs realtime --no-color --tail=50
  curl -i http://127.0.0.1:54340/api/health   # Expect 200 OK
  ```
- Health checks and JWT roundtrip:
  ```bash
  # From VPS
  curl -i http://127.0.0.1:5555/health   # Expect 200

  # Generate signed anon JWT and hit PostgREST (5512)
  SECRET=$(grep ^JWT_SECRET= .env | cut -d= -f2)
  HDR=$(printf '{"alg":"HS256","typ":"JWT"}' | openssl base64 -A | tr '+/' '-_' | tr -d '=')
  NOW=$(date +%s); EXP=$((NOW+3600))
  PL=$(printf '{"role":"anon","iss":"supabase","iat":%s,"exp":%s}' "$NOW" "$EXP" | openssl base64 -A | tr '+/' '-_' | tr -d '=')
  SIG=$(printf "%s.%s" "$HDR" "$PL" | openssl dgst -sha256 -hmac "$SECRET" -binary | openssl base64 -A | tr '+/' '-_' | tr -d '=')
  TOKEN="$HDR.$PL.$SIG"
  curl -i http://127.0.0.1:5512 -H "Authorization: Bearer $TOKEN"  # Expect 200 OpenAPI JSON
  ```

Notes:
- All service ports are bound to `127.0.0.1` and accessed via SSH tunnels until Nginx/Kong + TLS are configured.
- Ensure `GOTRUE_API_HOST=0.0.0.0`, `GOTRUE_API_PORT=9999`, and compose port mapping `127.0.0.1:${AUTH_PORT}:9999` exist.

## 4) Minimal Inventory Stack Scaffold (no template)

If a stack template is not available, create a minimal scaffold to validate DB connectivity and prepare for Supabase services later.

### Steps (on the VPS)

1. Create Inventory directory and files
   ```bash
   mkdir -p ~/supabase-docker/inventory
   cd ~/supabase-docker/inventory
   
   # .env — use the same DB password from shared Postgres
   cat > .env << 'EOF'
   PROJECT_NAME=inventory
   TIMEZONE=UTC
   DB_HOST=shared_postgres
   DB_PORT=5432
   DB_USER=supabase
   DB_PASSWORD=change_this_to_a_strong_password
   DB_NAME=inventory_db
   EOF
   
   # docker-compose.yml — provides Adminer for quick DB validation (local-only)
   cat > docker-compose.yml << 'EOF'
   services:
     adminer:
       image: adminer:4
       container_name: inventory_adminer
       environment:
         - ADMINER_DEFAULT_SERVER=${DB_HOST}
       ports:
         - "127.0.0.1:8080:8080"  # access via SSH tunnel only
       restart: unless-stopped
       networks:
         - supabase_shared
   
   networks:
     supabase_shared:
       external: true
   EOF
   ```

2. Start and verify
   ```bash
   cd ~/supabase-docker/inventory
   docker compose up -d
   docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
   ```

3. Test DB via Adminer (optional)
   - From your workstation, open an SSH tunnel:
     ```bash
     ssh -L 8080:127.0.0.1:8080 supauser@<server_ip>
     ```
   - In a browser, go to http://127.0.0.1:8080 and log in:
     - System: PostgreSQL
     - Server: `shared_postgres`
     - Username: `supabase`
     - Password: the shared Postgres password
     - Database: `inventory_db`

4. Next (later): replace Adminer with real Supabase services, configure environment (JWT/keys/ports), and expose via Nginx + TLS.

---

## 5) Minimal Moving Stack Scaffold (no template)

Mirror the Inventory scaffold to validate the `moving_db` connection.

### Steps (on the VPS)

1. Create Moving directory and files
   ```bash
   mkdir -p ~/supabase-docker/moving
   cd ~/supabase-docker/moving
   
   # .env — same DB password as shared Postgres
   cat > .env << 'EOF'
   PROJECT_NAME=moving
   TIMEZONE=UTC
   DB_HOST=shared_postgres
   DB_PORT=5432
   DB_USER=supabase
   DB_PASSWORD=change_this_to_a_strong_password
   DB_NAME=moving_db
   EOF
   
   # docker-compose.yml — Adminer on a different LOCAL port
   cat > docker-compose.yml << 'EOF'
   services:
     adminer:
       image: adminer:4
       container_name: moving_adminer
       environment:
         - ADMINER_DEFAULT_SERVER=${DB_HOST}
       ports:
         - "127.0.0.1:8081:8080"  # host:container
       restart: unless-stopped
       networks:
         - supabase_shared
   
   networks:
     supabase_shared:
       external: true
   EOF
   ```

2. Start and verify
   ```bash
   cd ~/supabase-docker/moving
   docker compose up -d
   docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}' | grep moving_adminer
   ```
   Expected: `127.0.0.1:8081->8080/tcp`.

3. Test via SSH tunnel from your workstation
   ```bash
   ssh -L 8081:127.0.0.1:8081 supauser@<server_ip>
   ```
   Open http://127.0.0.1:8081 and log in (System: PostgreSQL):
   - Server: `shared_postgres`
   - Username: `supabase`
   - Password: shared Postgres password
   - Database: `moving_db`

4. If you changed the compose and port still shows 8081->8081
   ```bash
   docker compose up -d --force-recreate
   # or
   docker compose down && docker compose up -d
   ```

---

## 3) Server Preparation Checklist

### Base System Hardening
- [ ] Update & upgrade packages
  ```bash
  sudo apt update && sudo apt upgrade -y
  sudo timedatectl set-timezone UTC
  ```
- [ ] Create non-root sudo user
  ```bash
  sudo adduser supauser
  sudo usermod -aG sudo supauser
  ```
- [ ] SSH hardening (`/etc/ssh/sshd_config`)
  - [ ] `PermitRootLogin no`
  - [ ] `PasswordAuthentication no`
  - [ ] Restart: `sudo systemctl restart ssh`
- [ ] UFW firewall
  ```bash
  sudo apt install ufw -y
  sudo ufw allow OpenSSH
  sudo ufw allow 80
  sudo ufw allow 443
  sudo ufw --force enable
  ```
- [ ] Fail2Ban
  ```bash
  sudo apt install fail2ban -y
  sudo systemctl enable --now fail2ban
  ```

---

## 4) Docker & Compose Setup

- [ ] Install Docker & Compose
  ```bash
  curl -fsSL https://get.docker.com | sh
  sudo usermod -aG docker $USER
  newgrp docker
  docker --version && docker compose version
  ```
- [ ] Project structure
  ```bash
  mkdir -p ~/supabase-docker && cd ~/supabase-docker
  mkdir -p ./inventory ./moving ./backups ./shared
  ```
- [ ] Generate strong secrets
  ```bash
  # Generate unique secrets for each project
  openssl rand -base64 32   # JWT_SECRET_INVENTORY
  openssl rand -base64 32   # JWT_SECRET_MOVING
  openssl rand -base64 32   # ANON_KEY_INVENTORY
  openssl rand -base64 32   # ANON_KEY_MOVING
  openssl rand -base64 32   # SERVICE_ROLE_KEY_INVENTORY
  openssl rand -base64 32   # SERVICE_ROLE_KEY_MOVING
  openssl rand -hex 24      # DB password
  ```

---

## 5) Shared Postgres Configuration

### Create shared environment file
- [ ] Create `shared/.env`
  ```env
  # Shared Postgres
  POSTGRES_USER=supabase
  POSTGRES_PASSWORD=change_me_strong_db_password
  POSTGRES_DB=postgres
  
  # Timezone
  TIMEZONE=UTC
  ```

### Shared Postgres Compose
- [ ] Create `shared/docker-compose.yml`
  ```yaml
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
  ```

- [ ] Create `shared/init-databases.sql`
  ```sql
  -- Create databases for each project
  CREATE DATABASE inventory_db;
  CREATE DATABASE moving_db;
  
  -- Create users for each project (optional, can use shared user)
  CREATE USER inventory_user WITH PASSWORD 'inventory_password';
  CREATE USER moving_user WITH PASSWORD 'moving_password';
  
  -- Grant permissions
  GRANT ALL PRIVILEGES ON DATABASE inventory_db TO inventory_user;
  GRANT ALL PRIVILEGES ON DATABASE moving_db TO moving_user;
  ```

---

## 6) Inventory Project Stack

- [ ] Create `inventory/.env`
  ```env
  # Project-specific settings
  PROJECT_NAME=inventory
  
  # Database (shared Postgres on Docker network)
  POSTGRES_HOST=shared_postgres
  POSTGRES_PORT=5432
  POSTGRES_DB=inventory_db
  POSTGRES_USER=supabase
  POSTGRES_PASSWORD=change_me_strong_db_password
  DATABASE_URL=postgres://${POSTGRES_USER}:${POSTGRES_PASSWORD}@${POSTGRES_HOST}:${POSTGRES_PORT}/${POSTGRES_DB}
  
  # Supabase
  JWT_SECRET=change_me_inventory_jwt_secret
  ANON_KEY=change_me_inventory_anon_key
  SERVICE_ROLE_KEY=change_me_inventory_service_role_key
  
  # API
  API_EXTERNAL_URL=https://api-inventory.yourdomain.com
  SUPABASE_PUBLIC_URL=https://api-inventory.yourdomain.com
  
  # Studio
  STUDIO_PORT=3001
  # Local-only bind ports (Nginx will proxy)
  KONG_HTTP_PORT=8000
  KONG_HTTPS_PORT=8443
  PGBOUNCER_PORT=6432
  ```

- [ ] Create `inventory/docker-compose.yml`
  ```yaml
  services:
    kong:
      image: kong:2.8-alpine
      environment:
        KONG_DATABASE: "off"
        KONG_DECLARATIVE_CONFIG: /var/lib/kong/kong.yml
        KONG_DNS_ORDER: LAST,A,CNAME
        KONG_PLUGINS: request-transformer,cors,key-auth,acl
      volumes:
        - ./kong.yml:/var/lib/kong/kong.yml
      ports:
        - "127.0.0.1:${KONG_HTTP_PORT}:8000"
        - "127.0.0.1:${KONG_HTTPS_PORT}:8443"
      restart: unless-stopped

    auth:
      image: supabase/gotrue:v2.99.0
      environment:
        GOTRUE_API_HOST: 0.0.0.0
        GOTRUE_API_PORT: 9999
        API_EXTERNAL_URL: ${API_EXTERNAL_URL}
        GOTRUE_DB_DRIVER: postgres
        GOTRUE_DB_DATABASE_URL: postgres://${POSTGRES_USER}:${POSTGRES_PASSWORD}@${POSTGRES_HOST}:${POSTGRES_PORT}/${POSTGRES_DB}?search_path=auth
        GOTRUE_SITE_URL: ${SUPABASE_PUBLIC_URL}
        GOTRUE_URI_ALLOW_LIST: ${SUPABASE_PUBLIC_URL}
        GOTRUE_JWT_SECRET: ${JWT_SECRET}
        GOTRUE_JWT_EXP: 3600
        GOTRUE_JWT_DEFAULT_GROUP_NAME: authenticated
        GOTRUE_JWT_ADMIN_ROLES: service_role
        GOTRUE_JWT_AUD: authenticated
        GOTRUE_JWT_DEFAULT_GROUP_NAME: authenticated
      restart: unless-stopped

    rest:
      image: postgrest/postgrest:v11.2.0
      environment:
        PGRST_DB_URI: postgres://${POSTGRES_USER}:${POSTGRES_PASSWORD}@${POSTGRES_HOST}:${POSTGRES_PORT}/${POSTGRES_DB}
        PGRST_DB_SCHEMAS: public
        PGRST_DB_ANON_ROLE: anon
        PGRST_JWT_SECRET: ${JWT_SECRET}
        PGRST_DB_USE_LEGACY_GUCS: "false"
      restart: unless-stopped

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
      command: >
        sh -c "/app/bin/migrate && /app/bin/realtime eval 'Realtime.Release.seeds(Realtime.Repo)' && /app/bin/server"
      restart: unless-stopped

    storage:
      image: supabase/storage-api:v0.40.4
      environment:
        ANON_KEY: ${ANON_KEY}
        SERVICE_KEY: ${SERVICE_ROLE_KEY}
        POSTGREST_URL: http://rest:3000
        PGRST_JWT_SECRET: ${JWT_SECRET}
        DATABASE_URL: postgres://${POSTGRES_USER}:${POSTGRES_PASSWORD}@${POSTGRES_HOST}:${POSTGRES_PORT}/${POSTGRES_DB}
        FILE_SIZE_LIMIT: 52428800
        STORAGE_BACKEND: file
        FILE_STORAGE_BACKEND_PATH: /var/lib/storage
        TENANT_ID: stub
        REGION: stub
        GLOBAL_S3_BUCKET: stub
      volumes:
        - storage_data:/var/lib/storage
      restart: unless-stopped

    edge-functions:
      image: supabase/edge-runtime:v1.22.4
      environment:
        JWT_SECRET: ${JWT_SECRET}
        SUPABASE_URL: ${API_EXTERNAL_URL}
        SUPABASE_ANON_KEY: ${ANON_KEY}
        SUPABASE_SERVICE_ROLE_KEY: ${SERVICE_ROLE_KEY}
      volumes:
        - ./functions:/home/deno/functions
      restart: unless-stopped

    studio:
      image: supabase/studio:20231123-233b56b
      environment:
        STUDIO_PG_META_URL: http://meta:8080
        POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
        DEFAULT_ORGANIZATION_NAME: "Inventory Project"
        DEFAULT_PROJECT_NAME: "Inventory"
        SUPABASE_URL: ${API_EXTERNAL_URL}
        SUPABASE_KEY: ${ANON_KEY}
        SUPABASE_SERVICE_KEY: ${SERVICE_ROLE_KEY}
      ports:
        - "127.0.0.1:${STUDIO_PORT}:3000"
      restart: unless-stopped

    meta:
      image: supabase/postgres-meta:v0.68.0
      environment:
        PG_META_PORT: 8080
        PG_META_DB_HOST: ${POSTGRES_HOST}
        PG_META_DB_PORT: ${POSTGRES_PORT}
        PG_META_DB_NAME: ${POSTGRES_DB}
        PG_META_DB_USER: ${POSTGRES_USER}
        PG_META_DB_PASSWORD: ${POSTGRES_PASSWORD}
      restart: unless-stopped

    pgbouncer:
      image: pgbouncer/pgbouncer:1.19.0
      environment:
        DATABASES_HOST: ${POSTGRES_HOST}
        DATABASES_PORT: ${POSTGRES_PORT}
        DATABASES_USER: ${POSTGRES_USER}
        DATABASES_PASSWORD: ${POSTGRES_PASSWORD}
        DATABASES_DBNAME: ${POSTGRES_DB}
        POOL_MODE: transaction
        SERVER_RESET_QUERY: DISCARD ALL
        MAX_CLIENT_CONN: 100
        DEFAULT_POOL_SIZE: 20
        MIN_POOL_SIZE: 5
        RESERVE_POOL_SIZE: 5
        SERVER_LIFETIME: 3600
        SERVER_IDLE_TIMEOUT: 600
        LOG_CONNECTIONS: 1
        LOG_DISCONNECTIONS: 1
        LOG_POOLER_ERRORS: 1
      ports:
        - "127.0.0.1:${PGBOUNCER_PORT}:5432"
      restart: unless-stopped

  volumes:
    storage_data:

  networks:
    default:
      external: true
      name: supabase_shared
  ```

---

## 7) Moving Project Stack

- [ ] Create `moving/.env` (similar to inventory but with different ports/keys)
  ```env
  # Project-specific settings
  PROJECT_NAME=moving
  
  # Database
  POSTGRES_HOST=postgres
  POSTGRES_PORT=5432
  POSTGRES_DB=moving_db
  POSTGRES_USER=supabase
  POSTGRES_PASSWORD=change_me_strong_db_password
  
  # Supabase
  JWT_SECRET=change_me_moving_jwt_secret
  ANON_KEY=change_me_moving_anon_key
  SERVICE_ROLE_KEY=change_me_moving_service_role_key
  
  # API
  API_EXTERNAL_URL=https://api-moving.yourdomain.com
  SUPABASE_PUBLIC_URL=https://api-moving.yourdomain.com
  
  # Studio
  STUDIO_PORT=3002
  
  # Kong
  KONG_HTTP_PORT=8100
  KONG_HTTPS_PORT=8543
  
  # PgBouncer
  PGBOUNCER_PORT=6433
  ```

- [ ] Create `moving/docker-compose.yml` (identical structure to inventory, different ports)

---

## 8) Nginx Configuration & TLS

- [ ] Install Nginx & Certbot
  ```bash
  sudo apt install nginx -y
  sudo apt install certbot python3-certbot-nginx -y
  ```

- [ ] Create Nginx sites
  ```bash
  # Inventory API
  sudo tee /etc/nginx/sites-available/api-inventory << 'EOF'
  server {
      listen 80;
      server_name api-inventory.yourdomain.com;
      
      location / {
          proxy_pass http://127.0.0.1:8000;
          proxy_http_version 1.1;
          proxy_set_header Host $host;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header X-Forwarded-Proto $scheme;
          proxy_set_header Upgrade $http_upgrade;
          proxy_set_header Connection "upgrade";
          proxy_read_timeout 300s;
          proxy_connect_timeout 60s;
          proxy_send_timeout 60s;
      }
      
      client_max_body_size 50m;
  }
  EOF
  
  # Inventory Studio
  sudo tee /etc/nginx/sites-available/studio-inventory << 'EOF'
  server {
      listen 80;
      server_name studio-inventory.yourdomain.com;
      
      location / {
          proxy_pass http://127.0.0.1:3001;
          proxy_http_version 1.1;
          proxy_set_header Host $host;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header X-Forwarded-Proto $scheme;
          proxy_set_header Upgrade $http_upgrade;
          proxy_set_header Connection "upgrade";
      }
  }
  EOF
  
  # Moving API
  sudo tee /etc/nginx/sites-available/api-moving << 'EOF'
  server {
      listen 80;
      server_name api-moving.yourdomain.com;
      
      location / {
          proxy_pass http://127.0.0.1:8100;
          proxy_http_version 1.1;
          proxy_set_header Host $host;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header X-Forwarded-Proto $scheme;
          proxy_set_header Upgrade $http_upgrade;
          proxy_set_header Connection "upgrade";
          proxy_read_timeout 300s;
          proxy_connect_timeout 60s;
          proxy_send_timeout 60s;
      }
      
      client_max_body_size 50m;
  }
  EOF
  
  # Moving Studio
  sudo tee /etc/nginx/sites-available/studio-moving << 'EOF'
  server {
      listen 80;
      server_name studio-moving.yourdomain.com;
      
      location / {
          proxy_pass http://127.0.0.1:3002;
          proxy_http_version 1.1;
          proxy_set_header Host $host;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header X-Forwarded-Proto $scheme;
          proxy_set_header Upgrade $http_upgrade;
          proxy_set_header Connection "upgrade";
      }
  }
  EOF
  ```

- [ ] Enable sites and get TLS certificates
  ```bash
  sudo ln -s /etc/nginx/sites-available/api-inventory /etc/nginx/sites-enabled/
  sudo ln -s /etc/nginx/sites-available/studio-inventory /etc/nginx/sites-enabled/
  sudo ln -s /etc/nginx/sites-available/api-moving /etc/nginx/sites-enabled/
  sudo ln -s /etc/nginx/sites-available/studio-moving /etc/nginx/sites-enabled/
  
  sudo nginx -t && sudo systemctl restart nginx
  
  # Get certificates
  sudo certbot --nginx -d api-inventory.yourdomain.com --redirect --agree-tos -m you@example.com
  sudo certbot --nginx -d studio-inventory.yourdomain.com --redirect --agree-tos -m you@example.com
  sudo certbot --nginx -d api-moving.yourdomain.com --redirect --agree-tos -m you@example.com
  sudo certbot --nginx -d studio-moving.yourdomain.com --redirect --agree-tos -m you@example.com
  ```

---

## 9) Deployment Sequence

1. **Start shared Postgres**
   ```bash
   cd ~/supabase-docker/shared
   docker compose up -d
   ```

2. **Create Docker network**
   ```bash
   docker network create supabase_shared
   ```

3. **Deploy inventory stack**
   ```bash
   cd ~/supabase-docker/inventory
   docker compose up -d
   ```

4. **Deploy moving stack**
   ```bash
   cd ~/supabase-docker/moving
   docker compose up -d
   ```

5. **Verify all services**
   ```bash
   docker ps
   curl -I https://api-inventory.yourdomain.com
   curl -I https://studio-inventory.yourdomain.com
   curl -I https://api-moving.yourdomain.com
   curl -I https://studio-moving.yourdomain.com
   ```

---

## 10) Backup Strategy

- [ ] Create backup script `~/supabase-docker/backups/backup.sh`
  ```bash
  #!/usr/bin/env bash
  set -e
  
  STAMP=$(date +'%Y%m%d_%H%M%S')
  BACKUP_DIR="$HOME/supabase-docker/backups"
  
  # Nightly database dumps
  docker exec supabase_shared-postgres-1 pg_dump -U supabase inventory_db > "$BACKUP_DIR/inventory_${STAMP}.sql"
  docker exec supabase_shared-postgres-1 pg_dump -U supabase moving_db > "$BACKUP_DIR/moving_${STAMP}.sql"
  
  # Weekly base backup (Sundays)
  if [ "$(date +%u)" -eq 7 ]; then
      docker exec supabase_shared-postgres-1 pg_basebackup -U supabase -D /tmp/basebackup_${STAMP} -Ft -z
      docker cp supabase_shared-postgres-1:/tmp/basebackup_${STAMP} "$BACKUP_DIR/"
  fi
  
  # Cleanup old backups (keep 14 days)
  find "$BACKUP_DIR" -type f -name "*.sql" -mtime +14 -delete
  find "$BACKUP_DIR" -type d -name "basebackup_*" -mtime +30 -delete
  
  # Optional: sync to object storage
  # rclone copy "$BACKUP_DIR" remote:supabase-backups/
  ```

- [ ] Set up cron job
  ```bash
  chmod +x ~/supabase-docker/backups/backup.sh
  (crontab -l 2>/dev/null; echo "0 2 * * * /home/supauser/supabase-docker/backups/backup.sh") | crontab -
  ```

---

## 11) Monitoring & Maintenance

- [ ] Install monitoring tools
  ```bash
  # Node exporter for system metrics
  wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz
  tar xvfz node_exporter-1.6.1.linux-amd64.tar.gz
  sudo mv node_exporter-1.6.1.linux-amd64/node_exporter /usr/local/bin/
  
  # Create systemd service
  sudo tee /etc/systemd/system/node_exporter.service << 'EOF'
  [Unit]
  Description=Node Exporter
  After=network.target
  
  [Service]
  User=supauser
  Group=supauser
  Type=simple
  ExecStart=/usr/local/bin/node_exporter
  
  [Install]
  WantedBy=multi-user.target
  EOF
  
  sudo systemctl enable --now node_exporter
  ```

- [ ] PostgreSQL monitoring
  ```bash
  # Add postgres_exporter to shared compose
  # Monitor key metrics: connections, query performance, disk usage
  ```

- [ ] Uptime monitoring
  ```bash
  # Set up external monitoring for all 4 endpoints
  # Alert on 5xx errors, certificate expiry, high response times
  ```

---

## 12) Integration with Encrypted Secrets

Following the established secure configuration pattern:

- [ ] Add Supabase secrets to `src/config/encryptedSecrets.ts`
  ```typescript
  supabase: {
    // Inventory project
    inventoryUrl: "encrypted_inventory_api_url",
    inventoryAnonKey: "encrypted_inventory_anon_key",
    inventoryServiceRoleKey: "encrypted_inventory_service_role_key",
    
    // Moving project  
    movingUrl: "encrypted_moving_api_url",
    movingAnonKey: "encrypted_moving_anon_key",
    movingServiceRoleKey: "encrypted_moving_service_role_key"
  }
  ```

- [ ] Update `SecureConfigManager.ts` interface
  ```typescript
  interface SupabaseSecrets {
    inventoryUrl: string;
    inventoryAnonKey: string;
    inventoryServiceRoleKey: string;
    movingUrl: string;
    movingAnonKey: string;
    movingServiceRoleKey: string;
  }
  
  interface SecretConfiguration {
    // ... existing
    supabase: SupabaseSecrets;
  }
  ```

- [ ] Create Supabase service hooks
  ```typescript
  // src/hooks/useSupabaseConfig.ts
  export function useInventorySupabase() {
    const { getSecret } = useSecureConfig();
    return {
      url: getSecret('supabase.inventoryUrl'),
      anonKey: getSecret('supabase.inventoryAnonKey'),
      serviceRoleKey: getSecret('supabase.inventoryServiceRoleKey')
    };
  }
  
  export function useMovingSupabase() {
    const { getSecret } = useSecureConfig();
    return {
      url: getSecret('supabase.movingUrl'),
      anonKey: getSecret('supabase.movingAnonKey'),
      serviceRoleKey: getSecret('supabase.movingServiceRoleKey')
    };
  }
  ```

---

## 13) Security Hardening

### Network Security
- [ ] UFW rules (only 80/443/22 open)
- [ ] Fail2Ban for SSH and HTTP
- [ ] Optional: IP allowlist for Studio access
- [ ] Rate limiting in Nginx

### Application Security
- [ ] Unique JWT secrets per project
- [ ] RLS policies on all tables
- [ ] Service role key only for server-side operations
- [ ] Regular security updates

### Data Protection
- [ ] Encrypted backups
- [ ] Secure key storage (never in logs/repos)
- [ ] Audit logging enabled
- [ ] GDPR compliance considerations

---

## 14) Troubleshooting Guide

### Common Issues
- **502 Bad Gateway**: Check container health, port conflicts
- **Database connection errors**: Verify Postgres is running, credentials correct
- **TLS certificate issues**: Check DNS, firewall, certbot logs
- **High memory usage**: Tune Postgres shared_buffers, work_mem
- **Slow queries**: Enable pg_stat_statements, add indexes

### Useful Commands
```bash
# Check all containers
docker ps -a

# View logs
docker compose -f shared/docker-compose.yml logs postgres
docker compose -f inventory/docker-compose.yml logs kong
docker compose -f moving/docker-compose.yml logs auth

# Database access
docker exec -it supabase_shared-postgres-1 psql -U supabase -d inventory_db

# Restart services
docker compose -f inventory/docker-compose.yml restart
```

---

## 15) Upgrade Procedure

### Supabase Component Updates
1. **Backup databases** before any upgrade
2. **Update image tags** in compose files
3. **Pull new images**: `docker compose pull`
4. **Restart services**: `docker compose up -d`
5. **Verify functionality** on all endpoints

### PostgreSQL Updates
1. **Full backup** (pg_dump + pg_basebackup)
2. **Test restore** on development environment
3. **Schedule maintenance window**
4. **Update Postgres image tag**
5. **Monitor for migration issues**

---

## 16) Cost Optimization

### Resource Management
- **Shared Postgres**: Reduces memory footprint vs separate instances
- **Connection pooling**: PgBouncer prevents connection exhaustion
- **Storage optimization**: Regular cleanup of old executions, logs
- **Monitoring**: Track resource usage, scale when needed

### Scaling Strategy
- **Vertical scaling**: Upgrade VPS when CPU/RAM limits reached
- **Horizontal scaling**: Move heavy project to dedicated VPS
- **Storage scaling**: Add object storage for large files

---

This deployment plan provides a production-ready, secure, and scalable Supabase self-hosting solution that integrates seamlessly with the existing encrypted configuration system and follows established security practices.
