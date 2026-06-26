# Specification: Authentication & Stateless Sessions

## 1. Context & Strategy

Authentication uses OAuth code flow with stateless application JWTs. Token revocation is immediate because every protected request checks the `revoked_tokens` table in D1.

## 2. Authentication Lifecycle

### 2.1 OAuth Code Exchange

1. The Angular frontend redirects the user to `/api/v1/auth/login?provider=<provider>`.
2. The Worker redirects the user to the selected third-party identity provider.
3. The identity provider redirects the user back to `/api/v1/auth/callback?code=<code>&state=<state>`.
4. The Worker exchanges the authorization code for identity information.
5. The Worker verifies the identity token using native Web Crypto APIs against the provider's JWKS, cached through standard HTTP cache.
6. The Worker performs an `INSERT OR IGNORE` into the `users` table.
7. The Worker signs an internal application JWT containing the application `user_id` and returns it to the client.

### 2.2 Application JWT Policy

Application JWTs must contain:

- `sub`: application user ID.
- `name`: display name.
- `provider`: identity provider name.

Tokens have a strict 2-hour lifespan. There is no refresh-token strategy for the PoC. Expiration requires a new OAuth code-flow handshake.

## 3. Session Middleware & D1 Revocation Check

Every protected route passes through a Hono middleware layer that performs two serial checks:

1. **Cryptographic Check:** Verifies the application JWT signature in memory.
2. **Revocation Check:** Executes a fast indexed query against the `revoked_tokens` table in D1. If the token hash is blacklisted, execution halts immediately and returns HTTP `401 Unauthorized`.

## 4. Session Revocation

When a user hits `POST /api/v1/auth/logout`, the system immediately adds the incoming JWT signature to the `revoked_tokens` table. The client simultaneously purges the token from local storage.

Logout remains safe to retry.

## 5. Bounded Revoked-Token Cleanup

No persistent background worker, cron job, or idle maintenance process is permitted.

Cleanup runs opportunistically as part of the centralized background queue processor (`processOpportunisticQueue(env, ctx)`) triggered via `ctx.waitUntil()` on every request:

```ts
// Executed as a step in the background queue processor
ctx.waitUntil(
  env.DB.prepare(
    "DELETE FROM revoked_tokens WHERE expires_at < strftime('%s', 'now') LIMIT 50"
  ).run()
);
```

The cleanup operation is strictly capped so it completes quickly within the request-time background budget and does not create idle cost.
