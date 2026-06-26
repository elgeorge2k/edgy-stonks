# Opportunistic Event-Driven Architecture — Implementation Plan

This plan outlines the task specification to transition the Edgy Stonks backend into an opportunistic, event-driven state machine. All heavy execution flows (computations, network queries, blockchain operations) are deferred to background queue execution triggered by client-driven requests via `ctx.waitUntil()`.

---

## 1. Architectural Blueprint: Opportunistic Processing

```
                       ┌──────────────────────┐
                       │   Angular Frontend   │
                       └──────────┬───────────┘
                                  │
                       API Request│ (Poll, Action, Asset Fetch, etc.)
                                  ▼
                       ┌──────────────────────┐
                       │    Hono API Layer    │
                       │                      │
                       │ 1. Write initial     │
                       │    state to D1 (NEW/ │
                       │    PENDING)          │
                       │ 2. Return HTTP response│
                       └──────────┬───────────┘
                                  │
                                  │ ctx.waitUntil()
                                  ▼
                       ┌──────────────────────┐
                       │ Opportunistic Queue  │
                       │      Processor       │
                       │                      │
                       │ - Check CPU buffer   │
                       │ - Pull pending tasks │
                       │ - Run viem, math,    │
                       │   D1 updates, etc.   │
                       └──────────────────────┘
```

---

## 2. Database Schema Updates (`database-schema.md` / wrangler migrations)

Update the relational schema to support robust tracking, task statuses, and retry parameters.

1. **Incorporate `revoked_tokens` table:**
   ```sql
   CREATE TABLE IF NOT EXISTS revoked_tokens (
       token_hash TEXT PRIMARY KEY,
       user_id TEXT NOT NULL,
       expires_at INTEGER NOT NULL,
       FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
   );
   CREATE INDEX IF NOT EXISTS idx_revoked_tokens_user_expires ON revoked_tokens(user_id, expires_at);
   ```

2. **Add Foreign Key Constraints & Indexes:**
   - Confirm all foreign keys (e.g., `ticker` on `bets`) are defined.
   - Index `bets.status` and `crypto_transactions.status` to make queue scanning highly performant.

3. **Enhance Task State Tables:**
   - In `bets`, add `attempts INTEGER NOT NULL DEFAULT 0` and `error_message TEXT`.
   - In `crypto_transactions`, add `attempts INTEGER NOT NULL DEFAULT 0` and `error_message TEXT`.

---

## 3. State-Machine Endpoint Contracts

### 3.1 Betting Flow
1. **`POST /api/v1/bets` (Step 1 Intention):**
   - Validates baseline fields (ledger, ticker, balance, collateral limit).
   - Deducts `collateral_amount_cents` from the user's selected balance.
   - Writes `bets` row with `status = 'NEW'`, returning the generated `bet_id` immediately with HTTP 202.
2. **`PUT /api/v1/bets/:id` (Step 2 Lock/Confirm):**
   - Updates the bet record state to allow the background task to execute confirmation.
   - Returns a status mapping showing state transitions (polling-friendly).
3. **`POST /api/v1/bets/:id/cashout` & `PUT /api/v1/bets/:id/cashout`:**
   - Mark the bet intention as requiring cashout processing.

### 3.2 Wallet flows (Deposits & Withdrawals)
1. **`POST /api/v1/wallet/deposit` & `PUT /api/v1/wallet/deposit/:id`:**
   - Initial write creates `crypto_transactions` with status `NEW`.
   - Immediate HTTP 202 response.
2. **`POST /api/v1/wallet/withdraw` & `PUT /api/v1/wallet/withdraw/:id`:**
   - Verifies the user has the required mETH balance and creates a `PENDING` withdrawal record.
   - Immediate HTTP 202 response.

---

## 4. The Opportunistic Queue Engine

Implement `processOpportunisticQueue(env, ctx)` as a central utility run asynchronously via `ctx.waitUntil()` on **every** API request.

### 4.1 Queue Processing Loop
- **CPU Time Check:** Enforce a runtime guard (e.g., maximum execution time / iteration cap) to gracefully terminate before reaching Workers limits.
- **Task Prioritization Order:**
  1. **Expired Token Cleanup:** Bounded SQL purge of expired JWTs.
  2. **Bet Resolution:** Process active cashout and entry transactions.
  3. **Deposit Verification:** Verify on-chain receipts via `viem`.
  4. **Withdrawal Dispatch:** Sign and broadcast pending transactions via `viem`.
  5. **Asset Hydration:** Perform stale-first price updates as defined in `market-sync.md`.

### 4.2 Security & Double-Spend Auditing
- **Deposits Validation:**
  1. Invoke `publicClient.getTransactionReceipt({ hash })` using `viem`.
  2. Verify that the destination wallet belongs to the platform.
  3. Verify the sender wallet address matches the user's verified wallet profile.
  4. Perform an atomic check: Ensure `tx_hash` is unique and does not already exist with `CONFIRMED` status.
- **Withdrawals Dispatch:**
  1. Use private key secrets `BASE_VAULT_PRIVATE_KEY` stored inside Cloudflare Worker Secrets.
  2. Use `viem` to sign and broadcast transaction.
  3. Update transaction status in D1 as asynchronous completion verification proceeds.

---

## 5. Validation Plan

1. **Unit Tests:**
   - Test math routines in isolation (leverage coefficient $L=50$, dynamic payout multipliers, and floor-truncation to integer cents).
2. **Integration Verification:**
   - Mock `viem` network calls and confirm state machines correctly transition from `NEW` / `PENDING` to `CONFIRMED` or `FAILED`.
   - Validate that queue CPU limits terminate processing loops gracefully.
