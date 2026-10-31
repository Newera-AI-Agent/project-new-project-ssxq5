# Ban-bypass audit — sub0 notes (read-only audit; no source files modified)

Repo: /home/runner/work/project-new-project-ssxq5/project-new-project-ssxq5/pollinations
Commit: 8c3fbfed3718bf622c0401c14445843d99fbdce7 (2026-10-31)

## Ban-check map (grep repo-wide for isUserBanned|assertNotBanned|banned_users|BANNED_USER)
- shared/auth/ban.ts:4-12 `isUserBanned(user)` -> `user.banned === true && (!user.banExpires || new Date(user.banExpires) > new Date())`
- shared/auth/ban.ts:1-2 `ACCOUNT_RESTRICTED_MESSAGE`
- shared/auth/api-key.ts:208-217 `assertNotBanned()` throws `BannedAccountError`
- shared/auth/api-key.ts:372 `assertNotBanned(user)` — inside `authenticateSessionToken` (352-375)
- shared/auth/api-key.ts:433 `assertNotBanned(row.user)`; :434 `assertNotBanned(row.byopOwner)` — inside `toActiveAuthResult` (426-466), reached by `loadActiveApiKeyAuthResult` (382-392)
- shared/auth/api-key-creation.ts:173 rejects when `isUserBanned(owner)` (OAuth client creation)
- enter.pollinations.ai/src/middleware/auth.ts:59 `assertNotBanned(result.user)` (better-auth session cookie); :94 maps BannedAccountError -> 403
- enter.pollinations.ai/src/routes/x402-keys.ts:93 `isUserBanned(holdingUser)`
- enter.pollinations.ai/src/routes/app-lookup.ts:28 `isUserBanned(user)` -> `{found:false}`
- enter.pollinations.ai/src/routes/stripe.ts:343, :482 `isUserBanned`
- enter.pollinations.ai/src/utils/stripe-billing/auto-top-up.ts:68, :140 `isUserBanned`
- (context only) gen.pollinations.ai/src/community-models.ts:100, gen.pollinations.ai/src/routes/e2b.ts:724

## Auth modes traced
1. API key (Authorization: Bearer / X-API-KEY / ?key=) -> authenticateApiKeyRequest (226) -> selectApiKeyRow (396) -> toActiveAuthResult (426) -> assertNotBanned(owner)+assertNotBanned(byopOwner). SAFE.
2. Session token `st_` -> authenticateSessionToken (352) -> verifySessionToken -> user row -> assertNotBanned at 372. SAFE.
3. Agent-run token `ag_` -> authenticateAgentRunToken (319) -> loadActiveApiKeyAuthResult(parentApiKeyId) at 332 -> toActiveAuthResult -> isUsable (431) + assertNotBanned (433). SAFE.
4. Better-auth session cookie -> middleware/auth.ts:59 — depends on better-auth returning user.banned/banExpires. user table columns: shared/db/better-auth.ts:33-35 (banned/banReason/banExpires). Confirm additionalFields exposure next.

## Open questions
- shared/auth/additional-fields.ts: are banned/banExpires/banReason exposed as better-auth additionalFields on the user model? (If not, session-cookie ban check is a no-op => finding.)
- enter routes/session-token.ts + shared/auth/session-token.ts: token mint path.
- enter routes/x402-keys.ts: full flow; is the x402 key usable at gen without a ban check?
- Direct `createAuth(...).api.getSession()` calls outside middleware/auth.ts: routes/referral.ts:16, routes/product-analytics.ts:35, routes/stripe.ts:329 & :525 => candidate bypasses.
- observability/endpoints/app_top_weekly_refresh.pipe:7,30 references a `banned_users` table — check whether a separate D1 table exists vs the `user.banned` column (divergence risk).
