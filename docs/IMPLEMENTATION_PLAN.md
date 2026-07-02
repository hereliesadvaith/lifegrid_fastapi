# Lifegrid Backend — FastAPI + Keycloak + Postgres + Valkey Implementation Plan

> Building the backend for the existing **Lifegrid Flutter app** (`../lifegrid_flutter`).
> Everything runs in **Docker** via a single `docker compose up`:
> **FastAPI** (API) · **Keycloak** (auth) · **Postgres** (database) · **Valkey** (cache).
> The Flutter app stays the source of truth for the domain — the backend mirrors its
> data design (metadata + EAV) and business rules (schema lock, graceful chart degradation).

---

## 1. Architecture

```
Flutter app ──(login: OIDC direct-grant / auth-code)──▶ Keycloak :8080
     │                                                       │
     └──(API calls, Bearer access-token)──▶ FastAPI :8000 ───┘  (verifies JWT via
                                                 │    │          Keycloak JWKS)
                                                 ▼    └────▶ Valkey :6379 (cache)
                                           Postgres :5432
```

- **Keycloak owns identities.** Users register/log in against the `lifegrid` realm.
  FastAPI never stores passwords or issues tokens — it only *validates* Keycloak-issued
  JWTs and reads the user's `sub` claim.
- **Ownership** is per Keycloak user: every `model` and `chart` row carries an
  `owner_sub` (Keycloak user UUID); all queries filter on it.
- **v1 login flow**: direct-access-grant (email + password posted to Keycloak's token
  endpoint) — simplest for the existing Flutter Settings/Profile screen. The
  auth-code+PKCE browser flow can replace it later with **zero backend changes**.

---

## 2. Docker Compose — the three services

```
lifegrid_fastapi/
  docker-compose.yml
  .env                      # all creds/names below come from here
  keycloak/
    realm-export.json       # auto-imported on first boot
  backend/
    Dockerfile
    requirements.txt
    app/ ...
```

| Service    | Image                            | Notes                                                                 |
|------------|----------------------------------|-----------------------------------------------------------------------|
| `postgres` | `postgres:16-alpine`             | Two databases: `lifegrid` (app data) and `keycloak` (Keycloak's own). Named volume for persistence. Healthcheck: `pg_isready`. |
| `keycloak` | `quay.io/keycloak/keycloak:26.x` | `start-dev` for now; backed by the same Postgres; `--import-realm` loads `keycloak/realm-export.json`. `depends_on: postgres (healthy)`. |
| `valkey`   | `valkey/valkey:8-alpine`         | Cache. No persistence needed (`--save ''`), memory-capped with `--maxmemory 128mb --maxmemory-policy allkeys-lru`. Healthcheck: `valkey-cli ping`. |
| `backend`  | built from `backend/Dockerfile`  | Uvicorn `--reload` + bind-mounted source for dev. `depends_on: postgres, keycloak, valkey`. |

**Postgres init**: a small init script (`/docker-entrypoint-initdb.d/`) creates the
second database (`keycloak`) on first boot.

**Realm export** (`realm-export.json`) defines up front, so no manual console clicking:
- Realm `lifegrid`, registration enabled.
- Public client `lifegrid-app` with *Direct Access Grants* enabled (for the Flutter
  password flow).
- Optionally a test user for development.

**Networking**: containers talk over the compose network (`http://keycloak:8080`,
`postgres:5432`); the host (Flutter dev, curl, browser) uses `localhost:8080` /
`localhost:8000`. Token `iss` mismatch between the two hostnames is a classic trap —
fix by setting `KC_HOSTNAME` (or validating issuer against a configured value).

---

## 3. Backend structure

```
backend/app/
  main.py          # FastAPI app, router registration, CORS (Flutter web/dev origins)
  config.py        # pydantic-settings: DATABASE_URL, VALKEY_URL, KEYCLOAK_URL, REALM, CLIENT_ID
  database.py      # SQLAlchemy 2.0 engine + session dependency
  cache.py         # Valkey client + get/set/invalidate helpers (see §6)
  models.py        # ORM: Model, Field, Record, FieldValue, Chart
  schemas.py       # Pydantic v2 request/response models
  auth.py          # get_current_user dependency: JWKS fetch (cached in Valkey) + JWT validation
  routers/
    models.py      # /models + nested /models/{id}/fields
    records.py     # /models/{id}/records
    charts.py      # /charts
```

**Dependencies** (`requirements.txt`): `fastapi`, `uvicorn[standard]`,
`sqlalchemy`, `psycopg[binary]`, `pydantic-settings`, `pyjwt[crypto]`, `httpx`,
`redis` (the official client — fully compatible with Valkey).

### auth.py — the only auth code we write

1. On first use, fetch `{KEYCLOAK_URL}/realms/lifegrid/protocol/openid-connect/certs`
   (JWKS) and cache the keys.
2. `get_current_user` dependency: read the `Authorization: Bearer` header, verify
   signature / expiry / issuer / audience with PyJWT, return the `sub` claim.
3. `401` on any failure. No user table, no password handling, no token issuing.

---

## 4. Database schema — mirror the Flutter sqflite schema

Same shape and column names as `lifegrid_flutter/lib/data/database.dart` (schema v6),
so client↔server mapping is 1:1. Additions vs. the Flutter schema are **bold**.

```sql
models       (id PK, **owner_sub TEXT NOT NULL**, name, position, created_at,
              **updated_at**)
fields       (id PK, model_id FK→models ON DELETE CASCADE, name,
              type,          -- 'STR' | 'INT' | 'FLOAT' | 'DATE' | 'BOOL'
              position)
records      (id PK, model_id FK→models ON DELETE CASCADE, created_at, updated_at)
field_values (id PK, record_id FK→records ON DELETE CASCADE,
              field_id FK→fields ON DELETE CASCADE, value TEXT)
charts       (id PK, **owner_sub TEXT NOT NULL**, model_id FK→models ON DELETE CASCADE,
              type,          -- 'pie' | 'todo' | 'bar' | 'line'
              group_field  FK→fields ON DELETE SET NULL,
              style, title, position, created_at,
              date_filter,   -- 'week' | 'month' | 'year' | NULL
              date_field   FK→fields ON DELETE SET NULL,
              sum_field    FK→fields ON DELETE SET NULL,
              label_field  FK→fields ON DELETE SET NULL,
              done_field   FK→fields ON DELETE SET NULL,
              tag_field    FK→fields ON DELETE SET NULL)
```

- Values stay canonical **TEXT** (EAV), exactly like the app: `BOOL` as `'1'`/`'0'`,
  numbers/dates as their string forms. Encoding/validation rules are ported from
  `lib/data/field_type.dart`.
- Indexes mirror the app: `fields(model_id)`, `records(model_id)`,
  `field_values(record_id)`, `field_values(field_id)`, `charts(model_id)` —
  plus `models(owner_sub)`, `charts(owner_sub)`.
- v1 creates tables with SQLAlchemy `create_all`; **Alembic** migrations are deferred.

---

## 5. API surface

All routes require a valid Bearer token; all data is scoped to the token's `sub`.

| Route                                        | Purpose |
|----------------------------------------------|---------|
| `GET /health`                                | liveness (no auth) |
| `GET /models`                                | list models with fields + record counts (mirrors `loadModels`) |
| `POST /models`                               | create model |
| `PATCH /models/{id}` / `DELETE /models/{id}` | rename / delete (cascade) |
| `POST /models/{id}/fields`                   | add field — **409 if the model has records** (schema lock) |
| `PATCH /models/{id}/fields/{fid}`            | rename field (allowed even when locked, like the app) |
| `DELETE /models/{id}/fields/{fid}`           | delete field — **409 if locked** |
| `GET /models/{id}/records`                   | list records, values as `{field_id: value}` |
| `POST /models/{id}/records`                  | create record — values validated against field types |
| `PATCH /records/{id}` / `DELETE /records/{id}` | update / delete record |
| `GET /charts` · `POST /charts`               | list / create chart descriptors |
| `PATCH /charts/{id}` / `DELETE /charts/{id}` | update (e.g. `date_filter`) / delete |

**Business rules enforced server-side** (ported from `ModelsRepository`):
- **Schema lock** — field add/delete rejected with `409` once the model has records
  (`SchemaLockedException` equivalent).
- **Type validation** — each posted value must satisfy its field's type
  (INT parses as int, DATE as ISO date, …) → `422` with the field name on failure.
- **Chart degradation** — deleted fields `SET NULL` on chart columns; charts are never
  deleted by a field deletion (only by model deletion, via CASCADE).
- **Cross-model guards** — a chart's field refs must belong to its model; a record's
  values must reference fields of its model.

---

## 6. Caching — Valkey

Valkey is a **read-through cache in front of Postgres** plus shared storage for auth
keys. It is always safe to flush — nothing lives only in the cache.

**What gets cached** (keys namespaced per user, JSON values):

| Key                              | Value                                | TTL | Invalidated by |
|----------------------------------|--------------------------------------|-----|----------------|
| `jwks:{realm}`                   | Keycloak JWKS (signing keys)         | 1 h | expiry only (re-fetch on unknown `kid`) |
| `models:{sub}`                   | full `GET /models` response          | 5 min | any model/field/record mutation by that user |
| `records:{sub}:{model_id}`       | `GET /models/{id}/records` response  | 5 min | record mutation on that model; model delete |
| `charts:{sub}`                   | `GET /charts` response               | 5 min | chart mutation; model delete |

**Pattern** — plain and explicit, no decorator magic for v1:
- Read: `cache.get(key)` → hit: return; miss: query Postgres, `SETEX`, return.
- Write: perform the mutation in Postgres, then `DELETE` the affected keys
  (invalidate, don't update — recompute on next read).
- `cache.py` exposes `get_json`, `set_json`, `invalidate(*keys)` and the key builders,
  so routers never touch raw Valkey commands.

**Rules**:
- TTLs are a safety net; explicit invalidation is the real mechanism.
- Cache failures must **never** fail a request — wrap Valkey calls so a connection
  error logs a warning and falls through to Postgres (degraded, not broken).
- `records` mutations also invalidate `models:{sub}` (record counts live there) and
  `charts` reads are computed client-side from records, so no chart-data caching needed.

---

## 7. Build order

Each step is independently verifiable before moving on.

1. **Compose skeleton** — `docker-compose.yml` + `.env` + postgres init script +
   `realm-export.json` + valkey. ✅ Verify: Keycloak console up on `:8080`, realm
   `lifegrid` imported, both databases exist, `valkey-cli ping` → `PONG`.
2. **Backend container** — Dockerfile, config, DB session, `create_all`, `/health`
   (reports Postgres + Valkey connectivity).
   ✅ Verify: `docker compose up` → `curl localhost:8000/health`, tables in Postgres.
3. **Auth dependency** — JWKS validation (keys cached in Valkey), protected test route.
   ✅ Verify: get a token via `curl` from Keycloak's token endpoint (direct grant),
   call the route with/without it (200 vs 401); `jwks:*` key visible in `valkey-cli`.
4. **Models + fields** — CRUD, ownership scoping, schema-lock rule.
   ✅ Verify: via `/docs` (Swagger) with a pasted token; second user can't see
   first user's models.
5. **Records** — CRUD with per-type value validation.
6. **Charts** — CRUD with SET NULL degradation + cross-model guards.
7. **Caching layer** — `cache.py` + read-through/invalidation on the list endpoints (§6).
   ✅ Verify: second `GET /models` is a cache hit (log/latency); a `POST` then `GET`
   returns fresh data; stopping the valkey container degrades gracefully (API still 200s).
8. **Flutter integration** — login screen posting to Keycloak's token endpoint,
   token storage/refresh, and an `ApiRepository` implementing the same interface as
   `ModelsRepository` so the UI layer is untouched.

---

## 8. Out of scope (deliberate, for v1)

- **Alembic migrations** — `create_all` is enough until the schema moves.
- **Keycloak production mode** — TLS, `start` (not `start-dev`), real hostname config.
- **Auth-code + PKCE flow in Flutter** — direct grant first; swapping later is
  client-only.
- **Offline-first sync / conflict resolution** — the hard problem. v1 policy:
  *server is source of truth when logged in*. The `updated_at` columns are already
  in place for a future sync protocol.
- **Valkey persistence / clustering / auth** — it's a pure cache on a private compose
  network; losing it costs only a recompute.
- **Profile photo upload, pagination, rate limiting.**
