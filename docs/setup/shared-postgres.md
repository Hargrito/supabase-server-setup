---
title: Shared Postgres Setup
summary: Steps to provision a shared PostgreSQL service via Docker Compose for multiple Supabase stacks.
lastUpdated: 2025-08-19
owners: ["@team"]
tags: ["postgres", "docker", "shared"]
---

# Shared Postgres Setup

This guide provisions a shared Postgres for multiple stacks.

## Files to create
- `~/supabase-docker/shared/.env`
- `~/supabase-docker/shared/docker-compose.yml`
- `~/supabase-docker/shared/init-databases.sql`

## .env
```env
POSTGRES_USER=supabase
POSTGRES_PASSWORD=<strong_password>
POSTGRES_DB=postgres
TIMEZONE=UTC
```

## docker-compose.yml
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

volumes:
  postgres_data:
```

## init-databases.sql
```sql
CREATE DATABASE inventory_db;
CREATE DATABASE moving_db;
-- Optional per-stack users
CREATE USER inventory_user WITH PASSWORD 'inventory_password';
CREATE USER moving_user WITH PASSWORD 'moving_password';
GRANT ALL PRIVILEGES ON DATABASE inventory_db TO inventory_user;
GRANT ALL PRIVILEGES ON DATABASE moving_db TO moving_user;
```

## Bring up the service
```bash
cd ~/supabase-docker/shared
docker compose up -d
```

## Verify connectivity
```bash
docker exec -it $(docker ps --format '{{.Names}}' | grep postgres) pg_isready -U supabase -d postgres
```
