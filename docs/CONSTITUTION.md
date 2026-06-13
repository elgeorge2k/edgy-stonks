# Project Constitution

## 1. Core Mission & Philosophy

Edgy Stonks is a casino-themed virtual financial betting platform driven by real-world market data. The user experience emphasizes fast, high-dopamine casino interactions while the underlying economics remain isolated inside the application's closed-loop wagering system.

## 2. Immutable Architectural Constraints

- **Absolute Zero Idle Cost:** The infrastructure must generate `$0` in operational or computational cost when no active users are interacting with the system. Persistent background workers, cron jobs, idle servers, and always-on processes are prohibited.
- **Bounded Request-Time Maintenance:** Cleanup work may run only as a bounded operation attached to normal request handling, for example with `ctx.waitUntil()`. Such work must be capped so it completes within the request lifecycle and must not create idle cost.
- **On-Demand Micro-Runtimes:** Compute workloads must rely on short-lived, stateless execution environments triggered by consumer demand.
- **Strict Interface Decoupling:** The UI layer and domain/API layer must remain isolated. Communication is restricted to stateless asynchronous web protocols such as JSON over HTTP.
- **Configurable Third-Party Endpoints:** Provider endpoints for market data and blockchain RPC must be injected through environment configuration or secrets. Documentation must not hardcode provider-specific URLs.

## 3. Domain and Data Boundaries

- **Domain-Driven Design:** Business logic, casino rules, betting math, exposure caps, and ADL behavior must live in the domain engine.
- **Database as Ledger:** Cloudflare D1 records state and enforces schema integrity. D1 must not own business rules or dictate betting outcomes.
- **Lazy Evaluation & Async Hydration:** Synchronous user execution paths must not be blocked by third-party external HTTP network requests. Market synchronization and external API hydration must happen asynchronously through non-blocking runtime lifecycles.
- **Dual Ledger Model:** `virtual_balance_cents` tracks internal non-withdrawable casino currency. `meth_balance_cents` tracks mETH operations and is the withdrawable settlement ledger.

## 4. Core Design Principles

- Keep documentation modular by single responsibility.
- Keep API contracts, schema, auth, betting, monetization, and market sync isolated in their owning specs.
- Prefer deterministic integer ledger accounting for user balances.
- Prefer Web Standard APIs and tree-shakeable libraries in Cloudflare Workers.
- Treat all external provider integrations as asynchronous dependencies, not user-path blockers.
