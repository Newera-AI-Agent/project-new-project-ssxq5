# Unauthenticated unmetered expensive-surface audit (scratch notes)

## Confirmed code evidence

### gen.pollinations.ai/src/index.ts
- L258 `.use("/account/*", edgeRateLimit)` — the only top-level edge limit.
- L268-275 `/quests/catalog` -> proxies to ENTER via fetchEnter, no limiter.
- L284 `.route("/docs", createDocsRoutes(app))` — no limiter.
- L296-299 `.get("/openapi.json", ... buildMergedOpenApiSpec(c, app) ...)` — no limiter.
- L322-324 `serveSite(request, env) ?? app.fetch(...)` — hosted-site serving runs before any limiter.

### gen.pollinations.ai/src/routes/docs.ts
- L1098-1119 buildMergedOpenApiSpec: Promise.all([getGenerationSchema, fetchEnterSchema, fetchMediaSchema]) + merge + injectSamples each request.
- L783-790 generation schema memoized (module-level promise) — but enter + media schemas are NOT.
- L824-839 fetchEnterSchema: c.env.ENTER.fetch('/api/docs/open-api/generate-schema') EVERY request (outbound #1; forwards client headers).
- L841-876 fetchMediaSchema: fetch('https://media.pollinations.ai/openapi.json') EVERY request (outbound #2).
- L1130-1232 all docs routes unauthenticated; authenticateDocsRequest (L29/L500) is client-side JS only.

### shared/rate-limit.ts
- L27-50 per-key SHA-256 + per-IP limits; `!byIp` counts per IP only when cf-connecting-ip present (L45-47); 429 + Retry-After: 10 (L52-59).

### shared/client-ip.ts
- L3-6 getRealClientIp = c.req.header('cf-connecting-ip') || 'unknown' — no XFF/True-Client-IP.

### enter.pollinations.ai/src/index.ts
- L55 `.use("/api/*", rateLimit({ byIp: false }))`.
- L56 `.route("/api/docs", createDocsRoutes(api))`.

### enter.pollinations.ai/src/routes/docs.ts
- L17-28 `/open-api/generate-schema` regenerates the enter OpenAPI spec per request.

### media.pollinations.ai/src/index.ts
- L1206-1233 `/openapi.json` regenerates + re-serializes the media spec per request; no limiter; no Cache-Control.
- L316-322 keyCheckRateLimit = rateLimit({ byIp: true }).

## Remaining TODO
- gen/src/sites.ts; gen cache middleware; wrangler.toml limit values; PollenRateLimiter/CommunityModelRateLimiter; media GET route auth; enter/src/api.ts public routes; XFF repo-wide grep; OpenAPI JSON size estimate; PoC test design.
