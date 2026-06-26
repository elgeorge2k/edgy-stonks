# Core API Contract & REST Specifications

The platform enforces a lean REST contract between the Angular frontend and the Hono backend. Filtering and pagination rely on query parameters. Transactional safety relies on idempotent HTTP verbs.

---

## 1. Idempotent State-Machine Endpoint Catalog

To satisfy edge runtime constraints (such as the 50ms CPU limit) and ensure immediate client responsiveness, mutations follow a non-blocking **Opportunistic Event-Driven State Machine** pattern:

1. **POST (Intention):** Step 1 initiates the workflow, validates basic parameters, deducts preliminary balances, and inserts a `NEW` or `PENDING` state row in D1. Returns an immediate HTTP `202 Accepted` response with the generated tracking ID.
2. **PUT (Trigger / Transition):** Step 2 transitions state tags to notify the background queue processor that the item is ready for action. It does not perform heavy calculations or external calls synchronously.
3. **Background Processing:** In the background, an opportunistic queue processor triggered on every request via `ctx.waitUntil()` processes pending records (performing `viem` smart contract verification, pricing checks, ADL calculation, etc.) and updates the records to `COMPLETED` or `FAILED`.
4. **Client Polling:** The frontend client polls the record's state using GET requests until status moves to `COMPLETED` or `FAILED`.

| Method | Endpoint | Payload / Query Params | Description |
| :--- | :--- | :--- | :--- |
| **GET** | `/api/v1/assets` | `?active=1&limit=50` | Retrieves tracked tickers, current prices, active status, and cache metadata. |
| **GET** | `/api/v1/assets/:ticker/history` | `?range=30d` | Retrieves historical OHLC data to fuel charts. |
| **GET** | `/api/v1/portfolios` | None, inferred from JWT context | Retrieves active open positions for the authenticated user. |
| **GET** | `/api/v1/bets` | `?status=COMPLETED&resolution_status=CASHED_OUT&limit=20` | Retrieves the user's resolved or liquidated bet history. |
| **POST** | `/api/v1/bets` | `{ ticker: "NVDA", bet_type: "LONG", collateral_amount_cents: 5000, balance_type: "METH" }` | Step 1: Creates a bet intention with status `NEW`, executes validation, and returns HTTP 202. |
| **PUT** | `/api/v1/bets/:id` | None | Step 2: Confirms the bet, transitions status to `PENDING` to queue it for the background processor, and returns HTTP 202. |
| **POST** | `/api/v1/bets/:id/cashout` | None | Initiates manual closeout intention and sets cashout status to `NEW` (HTTP 202). |
| **PUT** | `/api/v1/bets/:id/cashout` | None | Transitions closeout status to `PENDING` to queue it for the background processor (HTTP 202). |
| **GET** | `/api/v1/wallet/transactions` | `?limit=10` | Retrieves the ledger history of crypto deposits and withdrawals. |
| **POST** | `/api/v1/wallet/deposit` | `{ tx_hash: "0x..." }` | Step 1: Initiates inbound transaction tracking and sets status to `NEW` (HTTP 202). |
| **PUT** | `/api/v1/wallet/deposit/:id` | None | Step 2: Transitions status to `PENDING` to queue on-chain confirmation checks for the background processor (HTTP 202). |
| **POST** | `/api/v1/wallet/withdraw` | None | Step 1: Initiates withdrawal intention, verifies mETH balances, and sets status to `NEW` (HTTP 202). |
| **PUT** | `/api/v1/wallet/withdraw/:id` | `{ amount_cents: 100000, address: "0x..." }` | Step 2: Locks the requested amount and transitions status to `PENDING` to queue the transaction signing/broadcasting for the background processor (HTTP 202). |
| **GET** | `/api/v1/auth/login` | `?provider=google` | Redirects the user to the selected OAuth2 identity provider. |
| **GET** | `/api/v1/auth/callback` | `?code=xyz&state=abc` | Exchanges the OAuth authorization code, registers or logs in the user, and issues the application JWT. |
| **POST** | `/api/v1/auth/logout` | None, inferred from JWT context | Adds the incoming JWT signature to the `revoked_tokens` table and clears client-side token state. |

---

## 2. Global Security, CORS, JWT Policy, and Rate Limiting

- **CORS Governance:** The Hono backend applies a strict global CORS middleware enforcing `Access-Control-Allow-Origin: <frontend_domain>`. The allowed domain is injected through Cloudflare Worker environment variables.
- **Stateless JWT Claims:** Authorization headers must pass a bearer JWT containing exactly `sub`, `name`, and `provider`.
- **TTL Policy:** Tokens have a strict 2-hour lifespan with no refresh-token strategy for the PoC.
- **Rate Limiting & Bot Defense:** API throttling and DDoS protection are offloaded to Cloudflare's Web Application Firewall and Bot Fight Mode at the DNS zone layer.
- **Content Security Policy:** The backend is a stateless JSON API and emits no HTML or executable client-side scripts. CSP is handled exclusively by the Angular frontend hosting configuration.
