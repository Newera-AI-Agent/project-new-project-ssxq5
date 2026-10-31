# Ban-enforcement bypass audit (pollinations) — working notes

## Ban model (verified)
- `shared/auth/ban.ts:1-16` — NO DB access. Pure predicate:
  - `isUserBanned(user)`: `user.banned === true && (!user.banExpires || new Date(user.banExpires) > now)`
  - `isBannedLoginError(error)`: error === "BANNED_USER" || "banned"
  - `ACCOUNT_RESTRICTED_MESSAGE` = "Account restricted. Contact billing@pollinations.ai"
- Columns (there is NO `banned_at` / `banned_until` / `is_banned`):
  - `shared/db/better-auth.ts:33` `banned: integer("banned", {mode:"boolean"}).default(false)`
  - `shared/db/better-auth.ts:34` `banReason: text("ban_reason")`
  - `shared/db/better-auth.ts:35` `banExpires: integer("ban_expires", {mode:"timestamp"})`
  - Table: `user` (better-auth schema), identity key = `user.id`
- Expiry semantics: ban is ACTIVE while `ban_expires` is NULL or in the future; EXPIRED (past timestamp) => not banned.

## Ban write path
- `enter.pollinations.ai/src/auth.ts:122-125` — Better Auth `admin()` plugin with `bannedUserMessage: ACCOUNT_RESTRICTED_MESSAGE`.
  Better Auth admin plugin writes `user.banned`, `user.banReason`, `user.banExpires` keyed by `user.id`.
- `enter.pollinations.ai/src/auth.ts:58-71` `isAdminUser()` excludes banned admins.
- No ban write logic in `packages/auth/src` (grep for ban|Ban|BAN => no matches) and none in enter/src/routes/admin.ts.

## Generation-side ban checks (verified)
- `shared/auth/api-key.ts:208-217` `assertNotBanned()` -> throws `BannedAccountError` (class at :64-69).
- API key path: `shared/auth/api-key.ts:226-239` `authenticateApiKeyRequest` -> `selectApiKeyRow` (:396-424)
  JOIN `apikey.referenceId = user.id` (:416) -> `toActiveAuthResult` -> `assertNotBanned(row.user)` (:433) + BYOP owner (:434).
- Session-token path: `shared/auth/api-key.ts:354-375` `authenticateSessionToken` -> `assertNotBanned(user)` (:372).
- Agent-run token path: `shared/auth/api-key.ts:~320-350` `authenticateAgentRunToken` -> `loadActiveApiKeyAuthResult` (:382-392) -> `toActiveAuthResult` -> :433.
- `gen.pollinations.ai/src/middleware/auth.ts:118-140` `auth()` middleware: calls `authenticateApiKeyRequest`, converts
  `BannedAccountError` -> HTTP 403 (:127-137). => every request through `auth()` re-checks the ban from D1 (no long-lived cache of ban state; only INVALID keys are cached, 300s, api-key.ts:219-221).
- `gen.pollinations.ai/src/middleware/auth.ts:142-150` `authFromSnapshot()` — NO ban check (internal only).
  - used at `gen.pollinations.ai/src/utils/execute-generation.ts:65` (Durable-Object handoff, job.auth snapshot)
  - used at `gen.pollinations.ai/src/routes/e2b.ts:799`

## Generation entrypoints mapped
- `gen.pollinations.ai/src/routes/proxy.ts:429-443` model-list paths use `auth()`; `:685` `.use(auth())` gates all generation routes
  below (image/text/video/audio/embeddings/3d/realtime). `:682-683` `/register` is BEFORE auth() (community model registration).
- `gen.pollinations.ai/src/routes/mcp.ts:176` `.use("/mcp/:serverId", auth(), frontendKeyRateLimit)`; `:178` requireUser.
- `gen.pollinations.ai/src/routes/audio.ts:4347` `.use("*", auth(), frontendKeyRateLimit, balance)`.
- `gen.pollinations.ai/src/routes/stem-separation.ts:28` `.use("/alpha/audio/stem-separation", auth(), frontendKeyRateLimit, balance)`.
- `gen.pollinations.ai/src/routes/jobs.ts:57` `.use("/v1/jobs/*", auth())`.
- `gen.pollinations.ai/src/routes/e2b.ts:707` sandboxApi(false, edgeRateLimit, auth()).
- `gen.pollinations.ai/src/index.ts:237` generationHandoffRoutes (internal /internal/generation/:id, DO-mediated).
- `gen.pollinations.ai/src/index.ts:259-265,268-275` /account/* and /quests/catalog proxied to enter.
- `gen.pollinations.ai/src/index.ts:323-324` `serveSite(request, env) ?? app.fetch(...)` => hosted sites served BEFORE the app (need sites.ts review).

## Still to verify
- x402/prepaid 'pollen' key path (enter/src/routes/x402-keys.ts, services/x402-key-purchases.ts) -> how identity resolves, ban check.
- shared/billing/balance.ts + deduction.ts (balance middleware ordering vs ban).
- shared/auth/api-key-creation.ts:173 (ban check at key creation only? line 173 `if (!owner || isUserBanned(owner)) rejectInvalidClientId()`).
- gen/src/community-models.ts:72/100, gen/src/routes/e2b.ts:724 (owner ban filters).
- sites.ts (hosted sites) — served before app => which auth?
- shared/auth/session-token.ts, agent-run-token.ts details.
- packages/auth/src/server.ts (no ban logic found by grep).
