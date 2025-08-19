# Stack Template

Use this template to create a new Supabase stack that runs behind Nginx and shares the central Postgres service.

Files:
- `.env.example` — fill and copy to `.env` per stack
- `docker-compose.yml.example` — bring up services
- `kong.yml.example` — declarative Kong configuration for routes and plugins

Recommended port allocation:
- KONG_HTTP_PORT: 8000 + 100*(N-1)
- KONG_HTTPS_PORT: 8443 + 100*(N-1)
- STUDIO_PORT: 3000 + N
- PGBOUNCER_PORT: 6431 + N

Create DB in shared Postgres named `<stack>_db`. Ensure secrets are unique per stack.
