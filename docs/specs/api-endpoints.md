# Core API Contract & REST Specifications

The platform enforces a lean REST contract between the Angular frontend and the Hono backend. Filtering and pagination rely on query parameters. Transactional safety relies on idempotent HTTP verbs.

---

## 1. Idempotent Unified Endpoint Catalog

Every mutation flow for bets, deposits, and withdrawals follows a 2-step idempotent execution pattern:

1. **POST:** Step 1 launches intention, reserves context, checks baseline rules, generates a tracking ID, and sets state to `NEW`.
2. **PUT:** Step 2 locks execution, idempotently processes the action, mutates state to `COMPLETED` or `FAILED`, and commits to the ledger. Payload-less retries with the same ID return the same cached HTTP response without duplicate execution.

| Method | Endpoint | Payload / Query Params | Description |
| :--- | :--- | :--- | :--- |
| **GET** | `/api/v1/assets` | `?active=1&limit=50` | Retrieves tracked tickers, current prices, active status, and cache metadata. |
| **GET** | `/api/v1/assets/:ticker/history` | `?range=30d` | Retrieves historical OHLC data to fuel charts. |
| **GET** | `/api/v1/portfolios` | None, inferred from JWT context | Retrieves active open positions for the authenticated user. |
| **GET** | `/api/v1/bets` | `?status=COMPLETED&resolution_status=CASHED_OUT&limit=20` | Retrieves the user's resolved or liquidated bet history. |
| **POST** | `/api/v1/bets` | `{ ticker: "NVDA", bet_type: "LONG", collateral_amount_cents: 5000, balance_type: "METH" }` | Step 1 creates a bet intention, generates `bet_id`, applies exposure checks, and sets status to `NEW`. |
| **PUT** | `/api/v1/bets/:id` | None | Step 2 locks the execution price, confirms the bet, moves it to the active portfolio, and sets status to `COMPLETED`. |
| **POST** | `/api/v1/bets/:id/cashout` | None | Initiates manual closeout intention and sets cashout intent state to `NEW`. |
| **PUT** | `/api/v1/bets/:id/cashout` | None | Finalizes cashout, calculates final multipliers, credits the user balance, sets status to `COMPLETED`, and sets `resolution_status` to `CASHED_OUT`. |
| **GET** | `/api/v1/wallet/transactions` | `?limit=10` | Retrieves the ledger history of crypto deposits and withdrawals. |
| **POST** | `/api/v1/wallet/deposit` | `{ tx_hash: "0x..." }` | Step 1 initiates inbound transaction verification from Base L2 and sets status to `NEW`. |
| **PUT** | `/api/v1/wallet/deposit/:id` | None | Step 2 validates on-chain confirmation, mints mETH to the user profile, and sets status to `COMPLETED`. |
| **POST** | `/api/v1/wallet/withdraw` | None | Step 1 initiates withdrawal intention and validates that the user has enough mETH balance. |
| **PUT** | `/api/v1/wallet/withdraw/:id` | `{ amount_cents: 100000, address: "0x..." }` | Step 2 locks the requested amount, signs the transaction through `viem`, broadcasts to Base L2, and sets status to `COMPLETED`. |
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
