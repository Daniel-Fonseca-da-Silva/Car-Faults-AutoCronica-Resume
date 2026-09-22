# 02 · car-faults-api (NestJS backend)

[← Back to README](../README.md)

The core backend of Auto Crónica. It is the only data source for the web and the app, and the only component allowed to call the AI service.

- [Stack](#stack)
- [Module layout](#module-layout)
- [Bootstrap and global conventions](#bootstrap-and-global-conventions)
- [Endpoint reference](#endpoint-reference)
- [Cursor pagination](#cursor-pagination)
- [Rate limiting](#rate-limiting)
- [AI integration](#ai-integration)
- [Storage (Cloudflare R2)](#storage-cloudflare-r2)
- [Health check and logs](#health-check-and-logs)
- [Environment variables](#environment-variables)

---

## Stack

| Layer | Technology |
|---|---|
| Framework | NestJS 11 (Express), TypeScript 5 |
| ORM / DB | TypeORM 0.3 · PostgreSQL 16 |
| Cache | Redis 7 (`@nestjs/cache-manager` + Keyv, plus `ioredis` directly for counters) |
| Auth | Passport (Google OAuth20 + JWT), `google-auth-library` for mobile ID tokens |
| Validation | `class-validator` / `class-transformer` |
| API docs | Swagger (`/docs`, `/docs-json`) |
| Security | Helmet, configurable CORS, `@nestjs/throttler`, Cloudflare Turnstile |
| Storage | Cloudflare R2 via `@aws-sdk/client-s3` |
| Logging | `nestjs-pino` (JSON in production, `pino-pretty` in dev) |
| Health | `@nestjs/terminus` |
| Runtime | Node 24, `distroless/nodejs24` nonroot image |

## Module layout

```
src/
  main.ts                 # bootstrap: helmet, cookies, CORS, versioning, ValidationPipe, Swagger
  app.module.ts           # global ThrottlerGuard + every module
  common/                 # client (X-Client), cors, enums, pagination (cursor/keyset), throttler, slugify
  database/               # DataSource + TypeORM options
  redis/                  # ioredis client, cache-manager, health indicator, key prefixes
  logger/                 # pino-http
  health/                 # GET /v1/health
  auth/                   # Google OAuth (web), ID token (mobile), JWT, guards (Jwt, OptionalJwt, Admin, Google)
  users/                  # profile, stats, deletion (anonymization)
  vehicle-models/         # vehicle catalog
  known-issues/           # known issues + public "top faults" listing
  fixes/                  # fixes + votes
  lookups/                # cache → DB → AI orchestration, mobile rate limit
  ai/                     # lookup/translate providers (stub | http) + retry
  turnstile/              # Cloudflare Turnstile verification
  reviews/                # 1-5 star reviews
  comments/               # comments (+ R2 URL validator)
  reports/                # content reports
  activity-log/           # searches, issues viewed, favorites
  user-vehicles/          # garage
  storage/                # R2 uploads
  platform/               # public stats, top faults, catalog for the sitemap
  admin/                  # CRUD for vehicles/issues/fixes + moderation
  migrations/             # 19 TypeORM migrations
```

Every domain module follows the same pattern: `controller` → `service` → `repository` (a wrapper around TypeORM's `Repository`) → `entity`, with separate input and response DTOs. Responses **never** return entities directly: they always go through a `*ResponseDto`, so internal columns don't leak.

## Bootstrap and global conventions

In `src/main.ts`:

- **URI versioning**: every route lives under `/v1`.
- **Global `ValidationPipe`** with `whitelist`, `forbidNonWhitelisted` and `transform`. Unknown fields return `400`, and query strings are converted to the right types.
- **Helmet** (CSP off, since the API serves no HTML) and **cookie-parser**.
- **CORS** from `CORS_ORIGINS` (comma-separated list).
- **Swagger** at `/docs` with Bearer auth. It is the source of truth for schemas.
- **Global `ThrottlerGuard`** (`APP_GUARD`).

## Endpoint reference

Legend: 🌐 public · 🔓 optional session · 🔐 JWT required · 🛡️ JWT + admin required

### Auth: `/v1/auth` (`THROTTLE_AUTH_*` limit)

| Method | Route | | Description |
|---|---|---|---|
| GET | `/auth/google?state={locale}` | 🌐 | Starts OAuth; `state` comes back on the callback to build the web URL |
| GET | `/auth/google/callback` | 🌐 | Creates/links the user, sets the `access_token` cookie, creates an exchange code and redirects to `WEB_APP_URL/{locale}/auth/callback?code=…` |
| POST | `/auth/session/exchange` | 🌐 | `{ code }` → `{ accessToken }` (single use, 60 s) |
| POST | `/auth/google/mobile` | 🌐 | `{ idToken }` → `{ accessToken, user }`. Accepts both the web and the Android client ID as audience |
| POST | `/auth/logout` | 🌐 | Revokes the `jti` (Redis denylist) and clears the cookie → `204` |

**Account linking:** if the `googleId` already exists, the user signs into that account. If not, but the email exists, the `googleId` is linked to that account. Otherwise a new account is created.

### Lookup: `/v1/lookups` (`THROTTLE_LOOKUPS_*` limit)

| Method | Route | | Description |
|---|---|---|---|
| GET | `/lookups` | 🔓 | `brand, model, year (≥1900), engine, fuelType, doors? (1-6), language?`. Headers `x-turnstile-token` (web) or `X-Client: mobile`. May trigger the AI |
| GET | `/lookups/by-path` | 🌐 | `make, model, year, fuelType, engine, doors?, language?` as slugs. Read-only; `404` if it doesn't exist |

Response (`LookupResponseDto`):

```jsonc
{
  "vehicle": {
    "id": "uuid", "brand": "Ford", "model": "Fiesta", "name": "Fiesta Mk4",
    "yearFrom": 1999, "yearTo": 1999, "engine": "1.25", "doors": 5,
    "fuelType": "gasoline", "imageUrl": "https://{r2-public-domain}/vehicles/{uuid}.jpg",
    "techSpecs": { "power_hp": 75 }
  },
  "knownIssues": [
    {
      "id": "uuid",
      "title": "Heater control valve (HCV) failure",
      "description": "…",
      "severity": "medium",           // low | medium | high | critical
      "typicalKm": 80000,
      "sources": ["https://…"],
      "fixes": [
        { "id": "uuid", "summary": "…", "steps": "…", "estimatedCostEur": "90.00",
          "source": "ai", "likes": 0, "dislikes": 0 }
      ]
    }
  ]
}
```

Specific errors: `403 { code: "TURNSTILE_REQUIRED" }`, `429` (global throttler or mobile AI quota), `503` (AI service unavailable).

### Platform (public): `/v1/platform`

| Method | Route | Description | Cache |
|---|---|---|---|
| GET | `/platform/stats` | `{ reportsCount, vehiclesCount, faultsCount }` (`reportsCount` = total number of comments) | Redis |
| GET | `/platform/faults` | Known issues ranked by **number of comments**; filters `locale, brand, model, year, engine, fuelType, doors`; language fallback with `contentLocale` | Redis |
| GET | `/platform/vehicles` | Paginated catalog (used by the web's `sitemap.xml`) | None |

### Community

| Method | Route | | Description |
|---|---|---|---|
| GET | `/fixes?knownIssueId=` | 🔓 | Fixes with `likes`/`dislikes` and, with a session, `myVote` |
| POST · PATCH · DELETE | `/fixes`, `/fixes/:id` | 🔐 | `source = user` fixes (only the author can edit/delete). The current clients show the curated catalog and don't expose creation |
| POST | `/fixes/:id/vote` | 🔐 | `{ value: "like" \| "dislike" }` (creates or changes); you can't vote on your own fix (`403`) |
| DELETE | `/fixes/:id/vote` | 🔐 | Removes the vote |
| GET | `/reviews?knownIssueId=` | 🌐 | Paginated reviews |
| POST · PATCH · DELETE | `/reviews`, `/reviews/:id` | 🔐 | 1 per user and issue; only the author can edit/delete |
| GET | `/comments?knownIssueId=` | 🌐 | Paginated comments |
| POST · PATCH · DELETE | `/comments`, `/comments/:id` | 🔐 | `body` + `imageUrl?` (must belong to `R2_PUBLIC_BASE_URL`) |
| POST | `/reports` | 🔐 | `{ contentType: comment\|review, contentId, reason, details? (≤1000) }`; `409` if already reported |

`reason`: `spam`, `offensive`, `inappropriate_photo`, `harassment`, `other`.

### User

| Method | Route | | Description |
|---|---|---|---|
| GET | `/users/me` | 🔐 | Profile |
| GET | `/users/me/stats` | 🔐 | `searchesCount, defectsConsultedCount, savedVehiclesCount, votesCount, dislikesCount, favoritedVehiclesCount` (Redis cache) |
| PATCH | `/users/me` | 🔐 | `name?`, `avatarUrl?` |
| DELETE | `/users/me` | 🔐 | **Anonymizes and soft-deletes**: name, email, avatar and googleId are replaced/cleared |
| GET | `/user-vehicles` | 🔐 | Paginated garage |
| GET | `/user-vehicles/status?vehicleModelId=&year=` | 🔐 | Is the vehicle already in the garage? |
| GET · POST · PATCH · DELETE | `/user-vehicles[/:id]` | 🔐 | CRUD. With `vehicleModelId` it inherits catalog data; without it, `brand/model/engine` are required |
| POST | `/activity-logs` | 🔐 | Records `defect_consulted` (and other creatable types) |
| GET | `/activity-logs/favorites` | 🔐 | Paginated favorites |
| GET | `/activity-logs/favorites/:vehicleModelId` | 🔐 | Favorite status |
| DELETE | `/activity-logs/favorites/:vehicleModelId` | 🔐 | Removes a favorite |

> Favorites are stored as `activity_logs` rows of type `vehicle_favorite` (with the `year` in `metadata`), protected by a partial unique index. That way they reuse the activity infrastructure and feed the profile stats.

### Storage

| Method | Route | | Description |
|---|---|---|---|
| POST | `/storage/comment-images` | 🔐 | multipart `file`, JPEG/PNG/WebP, ≤ 5 MB → `{ url }` |
| POST | `/storage/vehicle-images` | 🛡️ | Same rules; catalog photo for a model |

### Admin: `/v1/admin` 🛡️

| Resource | Routes | Notes |
|---|---|---|
| Models | `GET /admin/vehicle-models` (filters `brand`, `model`), `GET /:id` (includes paginated issues), `POST`, `PATCH /:id`, `DELETE /:id` | Any write invalidates the model's lookup keys |
| Known issues | `GET /admin/known-issues`, `GET /:id`, `POST`, `PATCH /:id`, `DELETE /:id` | Same |
| Fixes | `GET /admin/fixes/:id`, `POST`, `PATCH /:id`, `DELETE /:id` | Same |
| Reports | `GET /admin/reports?status=`, `PATCH /:id` (`reviewed`/`dismissed`), `DELETE /:id/content` | Removing the content marks the report `reviewed`; the listing includes a preview and whether the content still exists |

There is no endpoint to promote administrators: the `admin` role is assigned directly in the database.

### Health

`GET /v1/health` 🌐 (not throttled): heap memory (`HEALTH_MEMORY_HEAP_LIMIT_BYTES`), DB ping and Redis ping.

## Cursor pagination

Every listing returns `{ items, nextCursor }`. To get the next page, send `?cursor={nextCursor}`. `nextCursor: null` means you're on the last page. The cursor is opaque (base64 of a resource-specific keyset), and an invalid cursor returns `400`.

| Endpoint | Default `limit` | Max |
|---|---:|---:|
| `/fixes`, `/reviews`, `/comments`, `/user-vehicles`, `/activity-logs/favorites` | 20 | 100 |
| `/admin/known-issues`, `/admin/vehicle-models`, `/admin/reports` | 20 | 100 |
| `/platform/faults` | 9 | 48 |
| `/platform/vehicles` | 50 | 200 |

## Rate limiting

| Name | Applies to | Variables |
|---|---|---|
| default | Every route (global guard) | `THROTTLE_TTL_MS`, `THROTTLE_LIMIT` |
| auth | `/auth/*` | `THROTTLE_AUTH_TTL_MS`, `THROTTLE_AUTH_LIMIT` |
| lookups | `/lookups/*` | `THROTTLE_LOOKUPS_TTL_MS`, `THROTTLE_LOOKUPS_LIMIT` |
| mobile AI | Only when the AI actually runs, with `X-Client: mobile`, per IP | `THROTTLE_AI_MOBILE_TTL_MS`, `THROTTLE_AI_MOBILE_LIMIT` |

## AI integration

```
src/ai/
  ai-lookup.provider.ts             # AiLookupProvider interface + AiLookupResult type (the contract)
  ai-translate.provider.ts          # AiTranslateProvider interface
  ai-*-provider.factory.ts          # picks stub | http; only http is allowed in production
  http-ai-lookup.provider.ts        # POST AI_API_URL with Bearer AI_API_KEY
  http-ai-translate.provider.ts     # POST AI_TRANSLATE_URL
  post-ai-with-retry.ts             # 1 retry for fast, transient failures
  stub-ai-*.provider.ts             # canned responses for dev/tests
  dto/ai-*-result.dto.ts            # defensive parsing/validation of the response
```

| Variable | Values |
|---|---|
| `AI_PROVIDER` | `stub` (default outside production) \| `http` (**required** with `NODE_ENV=production`, otherwise the app won't start) |
| `AI_API_URL` | e.g. `http://localhost:8000/lookup` (from Docker: `http://host.docker.internal:8000/lookup`) |
| `AI_TRANSLATE_URL` | e.g. `http://localhost:8000/translate` |
| `AI_API_KEY` | Same value as the car-faults-ai-api `API_KEY` |

The AI response is validated (`parseAiLookupResult`) before it is saved. A malformed payload never reaches the database.

## Storage (Cloudflare R2)

`R2StorageService` uses the S3 client pointed at `https://{R2_ACCOUNT_ID}.r2.cloudflarestorage.com`. After an upload it returns `R2_PUBLIC_BASE_URL/{key}`. Keys are generated by the server: `comments/{userId}/{uuid}.{ext}` or `vehicles/{uuid}.{ext}`. The same `R2_PUBLIC_BASE_URL` is used by the comment validator and by the web's `next/image` config.

## Health check and logs

- Structured logs via `pino-http`; JSON in production, ready for log aggregators.
- Every Redis failure is logged as `warn` and doesn't interrupt the request (fail-open).

## Environment variables

See `car-faults-api/.env.example`. Grouped:

| Group | Variables |
|---|---|
| App | `NODE_ENV`, `PORT`, `CORS_ORIGINS` |
| Postgres (Nest) | `DATABASE_HOST`, `DATABASE_PORT`, `DATABASE_USER`, `DATABASE_PASSWORD`, `DATABASE_NAME` |
| Postgres (compose) | `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB`, `POSTGRES_PORT` |
| Redis | `REDIS_HOST`, `REDIS_PORT`, `REDIS_USER?`, `REDIS_PASSWORD?`, `REDIS_USER_CACHE_TTL_MS`, `REDIS_LOOKUP_CACHE_TTL_MS`, `REDIS_USER_STATS_CACHE_TTL_MS`, `REDIS_PLATFORM_CACHE_TTL_MS` |
| JWT | `JWT_SECRET`, `JWT_EXPIRES_IN?` |
| Google | `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `GOOGLE_CALLBACK_URL`, `GOOGLE_ANDROID_CLIENT_ID` |
| Web / cookie | `WEB_APP_URL`, `COOKIE_SAME_SITE` (`lax`\|`strict`\|`none`), `COOKIE_SECURE` |
| AI | `AI_PROVIDER`, `AI_API_URL`, `AI_TRANSLATE_URL`, `AI_API_KEY` |
| Throttling | `THROTTLE_*` (see above) |
| Turnstile | `TURNSTILE_SECRET_KEY`, `TURNSTILE_ENABLED` (`false` only locally/in tests) |
| R2 | `R2_ACCOUNT_ID`, `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`, `R2_BUCKET_NAME`, `R2_PUBLIC_BASE_URL` |
| Health | `HEALTH_MEMORY_HEAP_LIMIT_BYTES` |

Setup, migrations and deployment: [08 · Development and deployment](08-development-and-deployment.md). Data model: [06 · Data model](06-data-model.md).
