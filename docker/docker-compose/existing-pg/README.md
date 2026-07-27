# Hindsight with an existing Postgres (non-superuser)

Use this compose file when you already run Postgres yourself (bare-metal or in
Docker) and want Hindsight to connect as a dedicated, non-superuser role rather
than the Postgres superuser.

## One-time database setup

Run these once, as a superuser, against your existing Postgres:

```sql
CREATE USER hindsight_user WITH PASSWORD 'change-me';
CREATE DATABASE hindsight OWNER hindsight_user;
\c hindsight
CREATE EXTENSION IF NOT EXISTS vector;
```

- Making `hindsight_user` the **owner** of the `hindsight` database is what lets
  it create tables later without extra grants: on Postgres 15+, the `public`
  schema is owned by the `pg_database_owner` pseudo-role, which implicitly
  includes whoever owns the current database.
- The superuser installs the `vector` extension once. Hindsight's migrations
  run `CREATE EXTENSION IF NOT EXISTS vector` on every startup, which is a
  permission-free no-op when the extension is already present -- so
  `hindsight_user` never needs extension-creation (superuser-equivalent)
  privileges.

If running via `docker exec` against a Postgres container:

```bash
docker exec -it <postgres-container> psql -U postgres -c "CREATE USER hindsight_user WITH PASSWORD 'change-me';"
docker exec -it <postgres-container> psql -U postgres -c "CREATE DATABASE hindsight OWNER hindsight_user;"
docker exec -it <postgres-container> psql -U postgres -d hindsight -c "CREATE EXTENSION IF NOT EXISTS vector;"
```

## Start Hindsight

```bash
export HINDSIGHT_API_DATABASE_URL="postgresql://hindsight_user:change-me@<postgres-host>:5432/hindsight"
export HINDSIGHT_API_LLM_API_KEY="your-llm-api-key"

docker compose -f docker/docker-compose/existing-pg/docker-compose.yaml up -d
```

Use the Postgres container's name as `<postgres-host>` when both containers
share a Docker network (see `POSTGRES_NETWORK` in `docker-compose.yaml`).
