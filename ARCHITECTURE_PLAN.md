# Architecture Separation Plan

## Purpose

This document captures the separation-of-interest strategy for the application. It defines the frontend, backend, communication, authentication, and database decisions so the team can revisit and update the plan as requirements evolve.

---

## Current State

Last updated: 2026-07-06.

The repository is a **monorepo** with three workspaces:

| Workspace | Stack | Role |
|-----------|-------|------|
| `frontend/` | Next.js 15, React 19 | Capture UI, overlays, chat, browser sample-bag CV |
| `backend/` | Express | Inspection pipeline, LLM orchestration, SOP/PDF, geometric lean |
| `shared/` | TypeScript (no runtime) | Schemas, purple-seed CV, sample-bag rating |

Key characteristics:

- UI and server logic are **separated**; the frontend proxies API calls through Next.js route handlers (`frontend/app/api/inspect/*`) to the Express backend on port 4000.
- Backend routes: `POST /inspect`, `POST /inspect/chat`, `POST /inspect/sample-bag`, `GET /health`.
- Shared validation lives in `shared/schemas/` and is imported by both sides.
- PostgreSQL is **optional** (`DATABASE_URL`) for audit logs and DB-backed API keys; the backend remains stateless for inspection payloads.
- **Sample bag counting** has two paths:
  - **Primary (UI):** browser CV via `shared/purpleSeed` — no image upload.
  - **Secondary (API):** LLM vision counter at `POST /inspect/sample-bag` with optional few-shot references.
- **Geometric lean gate:** optional second vision call during inspection measures inch-scale lean/overhang against SOP Appearance limits (2 in overhang, 5 in lean).

---

## High-Level Goal

Separate the application into distinct concerns:

1. **Frontend**: standalone UI application.
2. **Backend API Server**: standalone service.
3. **Communication**: CORS-enabled API with authentication.
4. **Database**: external service only if needed, with a preference for Postgres and minimal persistence.

The frontend should be deployable independently from the backend. The backend should be stateless wherever possible and should avoid in-memory persistence that retains data beyond the request lifetime.

---

## Target Architecture

### Frontend

- Standalone Next.js app (`frontend/`).
- Responsibilities:
  - capture and upload images/videos
  - display previews and defect overlays
  - run sample-bag purple-seed counting in the browser (`shared/purpleSeed`)
  - call backend API endpoints for inspection and chat
  - render inspection, sample-bag, and chat results
  - maintain local UI state only
  - manage upload state, file selection, progress, and result rendering
  - proxy backend requests server-side (inject API key via middleware)

### Backend

- Separate Express service (`backend/`).
- Responsibilities:
  - accept multipart form uploads and JSON requests
  - validate images, SOP text, inspection answers, and request payloads
  - perform PDF text extraction and SOP rule parsing
  - call the LLM inspection pipeline and chat flow
  - run geometric lean/overhang extraction and gate against SOP Appearance limits
  - serve an LLM-based sample bag counter API (optional path; UI uses browser CV)
  - normalize inspection results and defect geometry
  - provide chat follow-up responses
  - enforce rate limiting and authentication
  - support CORS for approved frontend origins
  - expose a health endpoint for readiness checks

### Communication

- REST API with JSON payloads.
- Endpoints:
  - `POST /inspect` — pallet/product vision inspection (+ optional `geometricLean` in response)
  - `POST /inspect/chat` — follow-up Q&A about a result
  - `POST /inspect/sample-bag` — LLM-based purple seed count (API / integration path)
  - `GET /health`
- Frontend proxy: Next.js route handlers at `/api/inspect`, `/api/inspect/chat`, `/api/inspect/sample-bag` forward to the backend; `middleware.ts` injects `BACKEND_API_KEY` when configured.
- CORS configuration:
  - allow only approved frontend origins
  - configure `Access-Control-Allow-Origin`
  - support `Access-Control-Allow-Headers` and `Access-Control-Allow-Methods`
- Authentication:
  - token-based API authentication
  - preferred options: static API key header, Bearer token, or lightweight JWT
  - frontend sends authentication credentials with every request (server-side on proxy; direct browser calls require CORS + key)

### Database / Persistence

- Avoid using in-process persistence in RAM.
- If persistence is required, use an external database service.
- Recommended database: PostgreSQL.
- Use the database for:
  - minimal audit/log records
  - optional auth configuration / API key storage
  - optional SOP or standards cache if required
- If persistence is not required, keep the backend stateless and process each request on the fly.
- Do not store large image or video payloads in the database unless explicitly needed.

---

## Feature Modules (post-split)

### Inspection pipeline

```
POST /inspect
  → PDF/SOP assembly (folder + uploads)
  → vision LLM per frame
  → normalize + aggregate (multi-frame video)
  → optional geometric lean gate (runLeanGeometryExtraction + applyGeometricLeanGate)
  → applySopRules
  → InspectResult + geometricLean + sopRuleChecks
```

### Sample bag counter

| Path | Where | Input | Output |
|------|-------|-------|--------|
| Browser CV (UI) | `frontend/lib/browser/runSampleBagCvDetection.ts` | `<img>` / canvas frame | `SampleBagCounterResult` + overlay markers |
| LLM API | `backend/routes/inspect/sampleBagRoute.ts` | multipart `image` | `{ result: SampleBagCounterResult }` |

Shared logic: `shared/purpleSeed/*` (CV), `shared/sampleBag/rating.ts`, `shared/schemas/sampleBagCounterResult.ts`.

Rating rules: **Perfect** (≥15 seeds), **Marginal** (10–14), **Bad** (≤9).

### Geometric lean gate

- Second vision call extracts pallet deck and load stack geometry.
- Inch measurements use known pallet footprints (54×40, 40×40).
- Thresholds from SOP Appearance (`sop/appearanceLimits.ts`); overridable via `INSPECT_SOP_MAX_OVERHANG_IN`, `INSPECT_SOP_MAX_LEAN_IN`.
- Disabled when `INSPECT_FAST_MODE=true` unless `INSPECT_GEOM_LEAN_ENABLED` overrides.

---

## Existing Code Migration Notes

Migration from the original single Next.js app is **complete**. Historical notes for reference:

- Inspection orchestration moved from `app/api/inspect/route.ts` → `backend/src/routes/inspect/route.ts`.
- Chat orchestration moved from `app/api/inspect/chat/route.ts` → `backend/src/routes/inspect/chat/route.ts`.
- Server-only modules live under `backend/src/` (`llm`, `pdf`, `sop`, `inspect`, `prompts`, `rateLimit`, `sampleBag`).
- UI-only code lives in `frontend/components/`, `frontend/app/`, and `frontend/lib/browser/`.
- Shared types and validation schemas live in `shared/`.
- LLM API keys and backend secrets stay out of frontend client bundles (`NEXT_PUBLIC_*` excluded for secrets).

---

## Practical Recommendation

- Keep the backend independent and stateless for inspection images.
- Keep the UI as a standalone client that sends requests (or runs CV locally for sample bags).
- Defer database persistence until there is a clear use case beyond audit/API keys.
- Use Postgres only for data that must outlive a single request and cannot be kept ephemeral.
- Prefer browser CV for sample bags in the UI (latency, privacy, offline potential); keep the LLM sample-bag endpoint for integrations that need model-based counting or few-shot references.

---

## Migration Roadmap

### Phase 0: Audit and scope

- inventory current frontend-only vs server-only modules.
- document current API contract for `POST /inspect` and `POST /inspect/chat`.
- identify environment and LLM configuration requirements.

### Phase 1: Define boundaries

- design the backend API contract and request payload shapes.
- decide which data should be persisted, if any.
- specify auth model and approved frontend origins.

### Phase 2: Extract backend service

- create a standalone server project.
- move backend logic out of `app/api`.
- preserve the existing inspection flow in the backend.

### Phase 3: Create standalone frontend

- extract UI components into a separate frontend application.
- switch API calls from internal Next.js routes to the new backend origin.
- confirm CORS policy and auth headers work correctly.

### Phase 4: Add authentication

- implement token/auth checks on the backend.
- update frontend to send auth credentials with requests.
- secure CORS to only approved frontend origins.

### Phase 5: Introduce database service (optional)

- provision a PostgreSQL instance externally.
- add only the minimal schema required.
- use the DB for logs, auth, or caching only if necessary.

### Phase 6: Separate deployment

- host frontend on static or edge hosting.
- host backend on a dedicated server or container.
- host Postgres as a separate external service if used.

---

## Separation Checklist

- [x] capture and formalize the `inspect` and `inspect/chat` API contracts (see README API section).
- [x] separate server-only code from frontend UI code.
- [x] extract LLM request, response parsing, and normalization into backend service modules.
- [x] add request validation and standardized JSON error responses.
- [x] add rate limiting and auth checks at the backend boundary.
- [x] implement CORS policy for frontend origins.
- [x] keep backend stateless unless persistence is clearly required.
- [x] evaluate Postgres only after audit/logging or auth storage needs are confirmed.
- [x] add sample bag counter (browser CV + optional LLM API).
- [x] add geometric lean/overhang gate to inspection pipeline.

---

## Implementation status

Last updated: 2026-07-06.

| Phase | Status | Notes |
|-------|--------|-------|
| 0 — Audit and scope | Done | Monorepo inventory complete |
| 1 — Define boundaries | Done | API contract in README; shared schemas in `shared/` |
| 2 — Extract backend service | Done | `backend/` Express server with inspect routes |
| 3 — Standalone frontend | Done | `frontend/` Next.js app; `/api/*` proxy via route handlers |
| 4 — Authentication | Done | `BACKEND_API_KEY`, CORS allowlist; see [SECURITY.md](./SECURITY.md) |
| 5 — Database (optional) | Done | Postgres audit logs + DB API keys; optional via `DATABASE_URL` |
| 6 — Separate deployment | Done | `docker compose` runs `frontend` + `backend` services |

### Recent feature work (2026)

| Feature | Status | Location |
|---------|--------|----------|
| Sample Bag tab (browser CV) | Done | `frontend/` + `shared/purpleSeed/` |
| LLM sample bag API | Done | `POST /inspect/sample-bag` |
| Geometric lean gate | Done | `applyGeometricLeanGate`, `runLeanGeometryExtraction` |
| SOP Appearance limits | Done | `sop/appearanceLimits.ts` |
| Purple seed calibration script | Done | `scripts/calibrate-purple-seeds.ts` |

---

## Concrete split plan

### Proposed code layout

- `frontend/`
  - Next.js UI
  - contains `components/`, `app/`, and browser-only helpers
  - contains shared type/schema package for client-side validation
  - `app/api/inspect/*` proxies to backend

- `backend/`
  - Node.js API server
  - contains API routes, request validation, LLM orchestration, PDF/SOP extraction, and normalization
  - controls CORS, auth, rate limiting, and error responses

- `shared/`
  - shared schema and type definitions
  - reusable helpers that are safe for both frontend and backend
  - `schemas/`: `InspectResult`, `defectOutlines`, `sampleBagCounterResult`, `normalizeRegions`
  - `purpleSeed/`: CV pipeline (pixel classify, segment, NMS, overlay markers)
  - `sampleBag/rating.ts`: Perfect / Marginal / Bad thresholds
  - `llm/detectHtmlResponse.ts`

### File/module ownership

Frontend-only:
- `lib/browser/*`
  - `runSampleBagCvDetection.ts`
  - `detectPurpleKernelOutlines.ts`
  - `purpleSeedOverlayCanvas.ts`
  - `handDrawnArrow.ts`
  - `sampleVideoFrames.ts`
- `lib/http/readJsonResponse.ts`
- `lib/inspectionQuestions.ts`
- `lib/backendProxy.ts`
- UI components under `components/`
- `app/page.tsx`, `app/layout.tsx`, `app/api/inspect/`, and other client entrypoints
- `middleware.ts`

Shared schema/types + CV:
- `schemas/inspectResult.ts`
- `schemas/defectOutlines.ts`
- `schemas/sampleBagCounterResult.ts`
- `schemas/normalizeRegions.ts`
- `purpleSeed/*`
- `sampleBag/rating.ts`
- `llm/detectHtmlResponse.ts`

Backend-only:
- `lib/llm/*` except shared `detectHtmlResponse.ts`
- `lib/pdf/extractText.ts`
- `lib/sop/*`
- `lib/inspect/*` (including `applyGeometricLeanGate`, `palletLeanMeasure`)
- `lib/prompts/*` (including `sampleBagCounter.ts`)
- `lib/sampleBag/*` (reference/rules cache for LLM counter)
- `lib/rateLimit/slidingWindow.ts`
- `lib/palletSizes.ts`
- `lib/smartsheet/*`
- `lib/db/*` (optional Postgres)

### Step-by-step split work

1. create a new `backend/` project or folder
   - `package.json`, `tsconfig.json`, `src/`, and `server.ts`
   - keep the existing `openai`, `zod`, `pdf-parse`, and server-only dependencies there

2. move API routes and server modules
   - move `app/api/inspect/route.ts` and `app/api/inspect/chat/route.ts` into backend route handlers
   - move `lib/llm`, `lib/pdf`, `lib/sop`, `lib/inspect`, `lib/prompts`, `lib/rateLimit`, `lib/palletSizes`, and `lib/smartsheet` into backend
   - leave shared schemas in a package accessible to both frontend and backend

3. build the backend API contract
   - `POST /inspect` accepts multipart form data with `image`, optional `inspectionAnswersJson`, `standardsText`, `productProfile`, `standardsVersion`, `standardsPdf`, and `supportingFile`
   - `POST /inspect/chat` accepts JSON with `result`, `pdfNotice`, `multiFrameNote`, and `messages`
   - `POST /inspect/sample-bag` accepts multipart form data with a single `image`
   - add `GET /health`
   - return consistent JSON payloads and `{ error: string }` on failure

4. keep frontend UI and HTTP client code in `frontend/`
   - update client fetches to `backend` API origin instead of `/api/inspect`
   - keep `readJsonResponse` and browser-only helpers in frontend
   - keep `components/` and `app/` in frontend

5. establish shared schema package
   - export `InspectResult`, defect models, and `inspectResultSchema`
   - use shared schema types in both frontend and backend to avoid drift
   - consider a symlinked package or monorepo workspace if desired

6. add auth and CORS in backend
   - validate an API key / bearer token on incoming requests
   - allow only approved frontend origins
   - enforce CORS headers for client requests

7. optional persistence later
   - add Postgres only if audit logs, API keys, or SOP caching are required
   - otherwise keep backend stateless and per-request

## Directory move checklist

- [x] create `frontend/` and move UI code there
  - move `app/` and `components/` into `frontend/`
  - move `public/` into `frontend/` if the frontend remains a standalone web app
  - keep browser-only helpers in `frontend/lib/browser` and `frontend/lib/http`

- [x] create `backend/` and move server logic there
  - move `app/api/inspect/route.ts` and `app/api/inspect/chat/route.ts` into `backend/src/routes`
  - move `lib/llm`, `lib/pdf`, `lib/sop`, `lib/inspect`, `lib/prompts`, `lib/rateLimit`, `lib/palletSizes`, and `lib/smartsheet` into `backend/src`
  - keep `lib/inspect/inspectConfig.ts` and server-only validation in backend

- [x] create `shared/` and move common schema/types there
  - move `lib/schemas/inspectResult.ts`, `lib/schemas/defectOutlines.ts`, and `lib/schemas/normalizeRegions.ts` into `shared/`
  - move `lib/llm/detectHtmlResponse.ts` into `shared/`
  - export shared types from `shared/index.ts`

- [x] keep frontend-only utilities with the frontend
  - leave `lib/inspectionQuestions.ts` in `frontend/lib`
  - leave `lib/browser` and `lib/http` under `frontend/lib`
  - ensure `frontend/tsconfig.json` path aliases include `shared`

- [x] preserve tests with their owner modules
  - move frontend tests with `frontend/`
  - move backend tests with `backend/`
  - move shared tests with `shared/`

- [x] update imports after moving files
  - replace `@/lib/...` imports with new front-end or backend paths
  - use shared package imports for schema types in both `frontend/` and `backend/`

- [x] create project roots for each side
  - add `frontend/package.json`, `frontend/tsconfig.json`
  - add `backend/package.json`, `backend/tsconfig.json`
  - optionally configure root workspaces for `frontend`, `backend`, and `shared`

### Target deliverables for first split

- a working `backend` server serving the same `inspect` and `inspect/chat` behavior
- frontend UI calling the new backend origin
- shared shapes for `InspectResult` and defect rendering
- server-side PDF extraction and LLM orchestration moved out of the Next app

---

## Known dependency risk

- Current package workspaces resolve `next@15.5.18` and `postcss@8.5.15`.
- Audit still reports a moderate `next`/`postcss` vulnerability in the nested `next` dependency graph.
- A safe workaround is to monitor stable Next.js releases and update when a non-vulnerable `postcss` dependency is published.
- Avoid using `npm audit fix --force` here because it would downgrade `next` to an incompatible legacy version.

---

## Future revision notes

- Update this document when API routes change or new endpoints are added.
- Re-evaluate the database decision if audit, user sessions, or persisted inspection history become required.
- Revisit auth method if stronger access control is needed (see [SECURITY.md](./SECURITY.md)).
- Maintain the separation as a core architectural principle for future feature work.
- If sample-bag UI requirements change (e.g. server-side LLM counting as default), document the chosen path and deprecate the unused one explicitly.
- Keep SOP Appearance limits in sync with PDFs under `SOP_SCAN_DIRECTORY` (see `.cursor/rules/sop-daily-sync.mdc`).
