# Security

This document describes the security model for the split frontend/backend architecture. See [ARCHITECTURE_PLAN.md](./ARCHITECTURE_PLAN.md) for the full migration roadmap and [README.md](./README.md) for endpoint details.

Last updated: 2026-07-06.

## Protected routes

The Express backend applies the same auth middleware to every route:

| Method | Path | Purpose |
|--------|------|---------|
| `POST` | `/inspect` | Pallet/product vision inspection |
| `POST` | `/inspect/chat` | Follow-up Q&A about a result |
| `POST` | `/inspect/sample-bag` | LLM-based sample bag counter (API / integration path) |
| `GET` | `/health` | Readiness check |

When `BACKEND_API_KEY` or database-backed API keys are configured, **all** of the above require valid credentials. There is no unauthenticated public API surface except when auth is explicitly disabled for local development.

The Sample Bag **UI tab** does not call these routes. It runs `shared/purpleSeed` CV entirely in the browser, so sample bag photos are not uploaded for that workflow.

---

## Authentication

The Express backend accepts an optional shared API key:

| Header | Format |
|--------|--------|
| `Authorization` | `Bearer <BACKEND_API_KEY>` |
| `X-API-Key` | `<BACKEND_API_KEY>` |

When `BACKEND_API_KEY` is **unset** and no active keys exist in Postgres, auth checks are skipped. This keeps local development simple.

When auth is **required** (env key set and/or active `api_keys` rows), every request to the protected routes above must include a matching key or the backend returns `401 Unauthorized`.

### Frontend proxy (recommended)

In production and local dev, the browser should call same-origin routes:

- `/api/inspect`
- `/api/inspect/chat`
- `/api/inspect/sample-bag`

Next.js **route handlers** under `frontend/app/api/inspect/` forward requests to the backend via `frontend/lib/backendProxy.ts`. `frontend/middleware.ts` injects `Authorization: Bearer …` from the server-side `BACKEND_API_KEY` env var on those paths.

The API key never appears in client-side JavaScript or `NEXT_PUBLIC_*` variables.

Proxy target defaults to `http://127.0.0.1:4000` (`BACKEND_ORIGIN`).

### Direct backend calls (optional)

If `NEXT_PUBLIC_BACKEND_ORIGIN` is set, the browser calls the backend directly. You must:

1. Add the frontend origin to `ALLOWED_ORIGINS`.
2. Set `BACKEND_API_KEY` on the backend (or use DB-backed keys).
3. Accept that exposing a browser-callable key is weaker than the proxy pattern.

Prefer the `/api` proxy for anything reachable from the public internet.

---

## PostgreSQL (audit logs and API keys)

When `DATABASE_URL` is set, the backend:

1. Runs migrations on startup (`audit_logs`, `api_keys` tables).
2. Writes an audit row for every `/inspect`, `/inspect/chat`, and `/inspect/sample-bag` response (metadata only — no images).
3. Validates API keys stored in `api_keys` (SHA-256 hash; raw key never stored).

When `DATABASE_URL` is **unset**, behavior falls back to env-only auth and no audit persistence.

### Auth with Postgres enabled

| Source | Behavior |
|--------|----------|
| `api_keys` table | Active keys accepted via `Authorization: Bearer` or `X-API-Key` |
| `BACKEND_API_KEY` env | Still accepted (break-glass / bootstrap) |
| Neither configured | Auth skipped (same as local dev without keys) |

On first startup with an empty `api_keys` table, `BACKEND_API_KEY` is copied into the database as `default (from BACKEND_API_KEY)` so existing deployments keep working.

### Creating team keys

```bash
# Start Postgres (Docker Compose includes a postgres service)
docker compose up -d postgres

# Set DATABASE_URL in .env.local, then:
npm run db:create-key -- --name "Team frontend"
npm run db:create-key -- --list
```

Store the printed key in your deployment secret store. Update `BACKEND_API_KEY` on the frontend proxy to match, or issue per-team keys.

### Audit log fields

Each row records: timestamp, route, client IP, API key name/id, HTTP status, pass/fail/chat/error outcome, confidence, defect count, summary text, product profile, standards version, image count, error message, and request duration. Images and PDFs are never stored.

Sample bag LLM requests log `product_profile: sample-bag-counter` and `image_count: 1` when successful.

Query example:

```sql
SELECT created_at, route, outcome, product_profile, summary, api_key_name
FROM audit_logs
ORDER BY created_at DESC
LIMIT 50;
```

---

## CORS

The backend uses an allowlist configured by `ALLOWED_ORIGINS` (comma-separated). Defaults:

- `http://localhost:3000`
- `http://127.0.0.1:3000`

Requests without an `Origin` header (server-to-server, curl, Next.js proxy) are allowed.

Allowed methods: `GET`, `POST`, `OPTIONS`.

Allowed headers: `Content-Type`, `Authorization`, `X-API-Key`.

---

## Rate limiting

Per-client IP rate limiting is enabled by default on all inspection endpoints:

| Route | Limited |
|-------|---------|
| `POST /inspect` | Yes |
| `POST /inspect/chat` | Yes |
| `POST /inspect/sample-bag` | Yes |
| `GET /health` | No |

| Variable | Default | Purpose |
|----------|---------|---------|
| `INSPECT_RATE_LIMIT_WINDOW_MS` | `60000` | Sliding window length |
| `INSPECT_RATE_LIMIT_MAX` | `30` | Max requests per window (shared across inspect routes) |

Exceeded limits return `429 Too Many Requests` with a `Retry-After` header.

Set `INSPECT_RATE_LIMIT_MAX=0` to disable (not recommended on public hosts).

The Sample Bag UI tab is not rate-limited by the backend because it does not call the backend.

---

## Upload and payload limits

| Endpoint | Constraint |
|----------|------------|
| `POST /inspect` | Up to 8 images, 10 MB each; optional PDFs (see `INSPECT_PDF_MAX_MB`, default 20 MB); supporting images up to 6 |
| `POST /inspect/sample-bag` | Single `image` field, 10 MB max |
| `POST /inspect/chat` | JSON body, 2 MB max (Express `json` middleware) |

Multipart uploads are parsed in memory/temp files per request and are not persisted after the response is sent.

---

## Secrets

Keep these server-side only:

- `MYGENASSIST_API_KEY` / `OPENAI_API_KEY`
- `BACKEND_API_KEY`
- `DATABASE_URL`
- Corporate proxy and TLS settings (`MYGENASSIST_HTTPS_PROXY`, `NODE_EXTRA_CA_CERTS`)

Safe to expose in the browser (non-secret toggles only):

- `NEXT_PUBLIC_BACKEND_ORIGIN` — backend URL when bypassing the proxy
- `NEXT_PUBLIC_SAMPLE_BAG_DEBUG` — enables client-side mask/contour debug UI only

Never commit `.env.local`. Use `.env.example` as the template.

---

## Data handling

### Inspection media (backend routes)

- Uploaded images and videos are processed per request and not written to Postgres.
- Formidable may use temporary files during multipart parsing; they are not retained after the handler completes.
- Vision LLM calls send image data to the configured gateway (`MYGENASSIST_API_BASE_URL` / `OPENAI_BASE_URL`). Treat the LLM provider as a data processor under your org's policy.

### Sample bag — two paths

| Path | Where data goes |
|------|-----------------|
| **UI tab (browser CV)** | Stays in the browser. No upload, no LLM call, no audit row. |
| **`POST /inspect/sample-bag` (LLM API)** | Image sent to backend → vision LLM. Optional few-shot reference images from `SAMPLE_SEED_REFERENCE_DIRECTORY` are attached server-side when `SAMPLE_BAG_FEW_SHOT_IMAGES=true`. |

### Server-side caches (memory only)

When configured, these are loaded at startup and held in process memory — not written to disk or Postgres:

- `SOP_SCAN_DIRECTORY` — PDF text for SOP rules
- `SAMPLE_SEED_REFERENCE_DIRECTORY` — reference photos for LLM sample bag few-shot
- Sample bag counter prompts/rules cache

Restart the backend after changing files in those directories.

### Postgres

Stores audit metadata and API key hashes only. No product images, sample bag photos, or PDF binaries.

---

## Dependency notes

See **Known dependency risk** in [ARCHITECTURE_PLAN.md](./ARCHITECTURE_PLAN.md) for current `next` / `postcss` audit status. Monitor stable Next.js releases rather than running `npm audit fix --force`.
