# Technical Specifications

`docs/SPECS.md` is the technical index for Edgy Stonks. It records global infrastructure decisions and links to domain-isolated implementation specifications.

---

## 1. Modular Documentation Index & Responsibilities

Refer to these documentation targets for global project rules, workflow boundaries, and domain-isolated specifications:

| Specification Document | Responsibility & Implementation Scope |
| --- | --- |
| `README.md` | High-level stakeholder overview, installation instructions, and project purpose. |
| [`docs/CONTRIBUTING.md`](CONTRIBUTING.md) | Developer onboarding, workflow guidelines, and automated validation gates. |
| [`docs/CONSTITUTION.md`](CONSTITUTION.md) | Immutable architectural boundaries, baseline system laws, and $0 idle-cost constraints. |
| [`docs/SPECS.md`](SPECS.md) | Core technical index, edge infrastructure topology, and global runtime decisions. |
| [`docs/specs/domain-model.md`](specs/domain-model.md) | **Strict Domain-Driven Design (DDD):** In-memory Aggregates, Entities, Value Objects, and Ubiquitous Language mappings. |
| [`docs/specs/market-sync.md`](specs/market-sync.md) | **Market Synchronization & Hydration:** Non-blocking in-memory `ctx.waitUntil()` asset price polling and cache rules. |
| [`docs/specs/database-schema.md`](specs/database-schema.md) | **Database Schema & Relational Design:** Cloudflare D1 relational layout, index optimization, and ledger tables. |
| [`docs/specs/auth-sessions.md`](specs/auth-sessions.md) | **Authentication & Stateless Sessions:** Federated OAuth flow, JWT session verification, and opportunistically pruned token revocation. |
| [`docs/specs/betting-engine.md`](specs/betting-engine.md) | **Betting Engine & Multiplier Mathematics:** Dynamic "Pays X:Y" pool-balancing calculations, exposure caps, and ADL logic. |
| [`docs/specs/monetization-engine.md`](specs/monetization-engine.md) | **Monetization & Internal Liquidity Engine:** Dual-ledger economics, mETH withdrawal gas structures, and platform revenue mechanics. |
| [`docs/specs/api-endpoints.md`](specs/api-endpoints.md) | **Core API Contract:** Idempotent REST interface communication schemas between Angular client and Hono runtime. |

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

## 3. Global Architectural Decisions

### 3.1 Market Synchronization

Market synchronization uses a hybrid model:

- Angular controls request frequency with visibility state.
- The Worker enforces a 5-minute stale-first cache rule.
- If cached asset data is fresh, the Worker returns it immediately.
- If cached asset data is stale, the Worker returns stale data with `X-Cache-Status: Hydrating` and hydrates D1 asynchronously.
- If no cached data exists, the Worker returns a cache-miss response and schedules hydration asynchronously.

See [specs/market-sync.md](specs/market-sync.md).

### 3.2 Authentication

The authoritative authentication flow is OAuth code flow:

1. `GET /api/v1/auth/login` redirects the user to the selected identity provider.
2. `GET /api/v1/auth/callback` exchanges the authorization code for an identity token.
3. The Worker verifies the identity token and issues an internal application JWT.
4. Protected routes verify the JWT and check D1 token revocation state.

See [specs/auth-sessions.md](specs/auth-sessions.md) and [specs/api-endpoints.md](specs/api-endpoints.md).

### 3.3 Currency and Ledger Model

The platform uses two ledgers:

- `virtual_balance_cents` tracks internal non-withdrawable casino currency.
- `meth_balance_cents` tracks mETH operations.
- Users can withdraw only mETH.
- Integer accounting is authoritative for balances and payouts.

See [specs/database-schema.md](specs/database-schema.md) and [specs/monetization-engine.md](specs/monetization-engine.md).

### 3.4 Betting Risk Controls

The domain betting engine owns exposure caps, dynamic payout scaling, and ADL behavior. D1 stores resulting ledger state and must not dictate business rules.

See [specs/betting-engine.md](specs/betting-engine.md).

### 3.5 Asset Curation

Asset curation does not expose public or administrative HTTP endpoints in the PoC backend. Operators manage ticker configuration directly in D1 through Cloudflare tooling:

- Wrangler D1 execution for scripted migrations and updates.
- Cloudflare Dashboard D1 console for manual SQL curation.


## 3.6 Opportunistic Event-Driven State Machine

To maximize the 50ms Edge CPU limit and strictly maintain our $0 idle cost constitution, the application avoids paid external queue services (like Cloudflare Queues). Instead, it implements a Transactional Outbox pattern utilizing D1 and `ctx.waitUntil()`. 

The core Hono API acts purely as a **State Machine Initiator**:
1. **Initiation:** Computationally heavy endpoints (e.g., placing bets, L2 withdrawals via viem) simply insert a `PENDING` record into the `background_tasks` table and immediately return an HTTP success response to the client.
2. **Opportunistic Execution:** On every organic incoming HTTP request to the API, a global middleware silently triggers a background execution via `ctx.waitUntil(processOpportunisticQueue())`.
3. **Processing:** The Worker utilizes its remaining allowed CPU execution time to asynchronously read and process a chunk of tasks from the `background_tasks` table, ensuring blazing-fast client responses while gracefully handling backend loads.

### 3.7 Strict Domain-Driven Design (DDD)

The application strictly adheres to Domain-Driven Design principles to protect business invariants. All financial math, invariant protection, and system rules are fully decoupled from both the database layer and the API routing layer. 

The Cloudflare Worker executes operations by instantiating pure, in-memory **Aggregate Roots** (such as `LiquidityPool` and `UserPortfolio`). The API controllers solely rely on these Aggregates to validate operations before committing any state changes to D1. Primitive obsession is systematically avoided through the use of strictly validated Value Objects.

See [specs/domain-model.md](specs/domain-model.md) for the complete Ubiquitous Language and object mapping.
---

## 4. Validation and Local Development Targets

Validation commands are target gates for implementation packages. They are not currently runnable in a spec-only checkout.

| Layer | Target Gate |
| --- | --- |
| API | `npm run lint`, `tsc --noEmit`, `npm run build` or `npx wrangler build`, and local Worker runtime through Wrangler/Miniflare. |
| Frontend | Angular lint/type-check/build gates and `ng build --configuration production`. |
| Local API Verification | Cloudflare Workers/Miniflare emulator at `http://localhost:8787` once the API package exists. |

See [docs/CONTRIBUTING.md](CONTRIBUTING.md).
