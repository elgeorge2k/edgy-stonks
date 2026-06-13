# Technical Specifications

`docs/SPECS.md` is the technical index for Edgy Stonks. It records global infrastructure decisions and links to domain-isolated implementation specifications.

---

## 1. Modular Documentation Responsibilities

| File | Responsibility |
| --- | --- |
| `README.md` | High-level stakeholder overview. |
| `docs/CONTRIBUTING.md` | Developer onboarding, workflow, and validation gates. |
| `docs/CONSTITUTION.md` | Immutable architectural boundaries and systemic constraints. |
| `docs/SPECS.md` | Technical index and global infrastructure decisions. |
| `docs/specs/market-sync.md` | Market-data hydration, cache behavior, and frontend sync cadence. |
| `docs/specs/database-schema.md` | Cloudflare D1 relational schema and ledger tables. |
| `docs/specs/auth-sessions.md` | OAuth flow, JWT session middleware, and token revocation. |
| `docs/specs/betting-engine.md` | Betting math, exposure caps, ADL, and position lifecycle. |
| `docs/specs/monetization-engine.md` | Dual-ledger economics, mETH withdrawal settlement, and revenue mechanics. |
| `docs/specs/api-endpoints.md` | REST contract between Angular and Hono. |

---

## 2. Concrete Technology Stack

To satisfy the principles defined in `docs/CONSTITUTION.md`, the current implementation target is:

- **Frontend Client:** Angular 17+ using Zoneless Signals and Angular Material, compiled as a static single-page application and deployed to Cloudflare Pages.
- **API Engine:** TypeScript using the Hono framework, deployed to Cloudflare Workers.
- **Data Persistence:** Cloudflare D1 as the primary SQLite ledger and cache store.
- **Blockchain Settlement:** Base L2 settlement through `viem` and a configurable RPC URL secret.
- **Market Data:** Massive.com provider data through a configurable endpoint configuration.
- **Runtime Model:** Cloudflare Workers with asynchronous hydration and no persistent idle compute.

---

## 3. Specifications Index

Refer to these targets when developing specific features:

- **Market Synchronization & Hydration:** [specs/market-sync.md](specs/market-sync.md)
- **Database Schema & Relational Design:** [specs/database-schema.md](specs/database-schema.md)
- **Authentication & Stateless Sessions:** [specs/auth-sessions.md](specs/auth-sessions.md)
- **Betting Engine & Multiplier Mathematics:** [specs/betting-engine.md](specs/betting-engine.md)
- **Monetization & Internal Liquidity Engine:** [specs/monetization-engine.md](specs/monetization-engine.md)
- **Core API Contract & REST Specifications:** [specs/api-endpoints.md](specs/api-endpoints.md)

---

## 4. Global Architectural Decisions

### 4.1 Market Synchronization

Market synchronization uses a hybrid model:

- Angular controls request frequency with visibility state.
- The Worker enforces a 5-minute stale-first cache rule.
- If cached asset data is fresh, the Worker returns it immediately.
- If cached asset data is stale, the Worker returns stale data with `X-Cache-Status: Hydrating` and hydrates D1 asynchronously through `ctx.waitUntil()`.
- If no cached data exists, the Worker returns a cache-miss response and schedules hydration asynchronously.

See [specs/market-sync.md](specs/market-sync.md).

### 4.2 Authentication

The authoritative authentication flow is OAuth code flow:

1. `GET /api/v1/auth/login` redirects the user to the selected identity provider.
2. `GET /api/v1/auth/callback` exchanges the authorization code for an identity token.
3. The Worker verifies the identity token and issues an internal application JWT.
4. Protected routes verify the JWT and check D1 token revocation state.

See [specs/auth-sessions.md](specs/auth-sessions.md) and [specs/api-endpoints.md](specs/api-endpoints.md).

### 4.3 Currency and Ledger Model

The platform uses two ledgers:

- `virtual_balance_cents` tracks internal non-withdrawable casino currency.
- `meth_balance_cents` tracks mETH operations.
- Users can withdraw only mETH.
- Integer accounting is authoritative for balances and payouts.

See [specs/database-schema.md](specs/database-schema.md) and [specs/monetization-engine.md](specs/monetization-engine.md).

### 4.4 Betting Risk Controls

The domain betting engine owns exposure caps, dynamic payout scaling, and ADL behavior. D1 stores resulting ledger state and must not dictate business rules.

See [specs/betting-engine.md](specs/betting-engine.md).

### 4.5 Asset Curation

Asset curation does not expose public or administrative HTTP endpoints in the PoC backend. Operators manage ticker configuration directly in D1 through Cloudflare tooling:

- Wrangler D1 execution for scripted migrations and updates.
- Cloudflare Dashboard D1 console for manual SQL curation.

---

## 5. Validation and Local Development Targets

Validation commands are target gates for implementation packages. They are not currently runnable in a spec-only checkout.

| Layer | Target Gate |
| --- | --- |
| API | `npm run lint`, `tsc --noEmit`, `npm run build` or `npx wrangler build`, and local Worker runtime through Wrangler/Miniflare. |
| Frontend | Angular lint/type-check/build gates and `ng build --configuration production`. |
| Local API Verification | Cloudflare Workers/Miniflare emulator at `http://localhost:8787` once the API package exists. |

See [docs/CONTRIBUTING.md](CONTRIBUTING.md).
