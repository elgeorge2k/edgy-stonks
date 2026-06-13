# Contributing Guidelines & AI Protocols — Edgy Stonks

This document defines the required development workflow for human engineers and AI coding agents contributing to the Edgy Stonks monorepo.

The repository is currently spec-first. Implementation commands in this file are target gates for when the `api/` and `frontend/` packages exist; they are not current runnable commands in a spec-only checkout.

---

## 1. Modular Architecture Blueprint

| File | Single Responsibility |
| --- | --- |
| `README.md` | High-level stakeholder overview. |
| `docs/CONTRIBUTING.md` | Developer onboarding, workflow, and validation gates. |
| `docs/CONSTITUTION.md` | Immutable architectural boundaries and systemic constraints. |
| `docs/SPECS.md` | Technical index, global infrastructure decisions, and deployment layout. |
| `docs/specs/market-sync.md` | Market-data hydration, cache behavior, and frontend sync cadence. |
| `docs/specs/database-schema.md` | Cloudflare D1 relational schema and ledger tables. |
| `docs/specs/auth-sessions.md` | OAuth flow, JWT session middleware, and token revocation. |
| `docs/specs/betting-engine.md` | Betting math, exposure caps, ADL, and position lifecycle. |
| `docs/specs/monetization-engine.md` | Dual-ledger economics, mETH withdrawal settlement, and revenue mechanics. |
| `docs/specs/api-endpoints.md` | REST contract between Angular and Hono. |

---

## 2. Required Reading Order

Before opening a pull request or generating implementation code, read the relevant files in this order:

1. `docs/CONSTITUTION.md` — immutable behavioral and architectural boundaries.
2. `docs/SPECS.md` — global technical index and infrastructure decisions.
3. The domain spec that owns the requested change.
4. `docs/specs/api-endpoints.md` when the change affects HTTP contracts.
5. `docs/specs/database-schema.md` when the change affects persistence or ledgers.
6. This file for workflow and validation expectations.

---

## 3. Spec Alignment Workflow

1. Confirm the feature is already represented in `docs/SPECS.md` or a domain spec.
2. If the feature is not represented, update the documentation first.
3. Keep implementation details inside the owning domain spec.
4. Keep global cross-cutting decisions in `docs/SPECS.md`.
5. Keep immutable constraints in `docs/CONSTITUTION.md`.
6. Do not add implementation code until the relevant spec is internally consistent.

---

## 4. Documentation Update Workflow

When changing documentation, preserve the modular architecture blueprint:

- Move stakeholder-facing summaries to `README.md`.
- Move developer workflow and validation gates to `docs/CONTRIBUTING.md`.
- Move immutable constraints to `docs/CONSTITUTION.md`.
- Move technical indexes and global decisions to `docs/SPECS.md`.
- Move domain implementation details to the matching `docs/specs/*.md` file.

Avoid duplicating implementation details across multiple files. If a detail is needed in more than one place, keep the authoritative version in the owning spec and summarize it elsewhere.

---

## 5. Target Validation Gates

The following gates are required once the implementation packages exist. They are not currently runnable in a spec-only checkout.

### 5.1 API Layer

Target stack: TypeScript, Hono, Cloudflare Workers.

| Gate | Target Command | Current Status |
| --- | --- | --- |
| Dependency install | `npm install` inside `api/` | Future; requires `api/package.json`. |
| Lint | `npm run lint` inside `api/` | Future; requires API package scripts. |
| Type check | `tsc --noEmit` inside `api/` | Future; requires TypeScript configuration. |
| Build | `npm run build` or `npx wrangler build` inside `api/` | Future; requires API package scripts. |
| Local runtime | `npm run dev` inside `api/` | Future; requires API package scripts. |

API bundles must remain compatible with Cloudflare Workers. The API build must not introduce unresolved Node.js-only dependencies such as `fs` or `path`, and the target Worker bundle should remain under 1MB where applicable.

### 5.2 Frontend Layer

Target stack: Angular 17+, Zoneless Signals, Angular Material.

| Gate | Target Command | Current Status |
| --- | --- | --- |
| Dependency install | `npm install` inside `frontend/` | Future; requires `frontend/package.json`. |
| Lint | `ng lint` or the project-defined Angular lint script | Future; requires Angular package scripts. |
| Type check | Angular production build type checking or `tsc --noEmit` if configured | Future; requires Angular configuration. |
| Production build | `ng build --configuration production` inside `frontend/` | Future; requires Angular configuration. |
| Local runtime | `ng serve` inside `frontend/` | Future; requires Angular configuration. |

Frontend components must follow the Zoneless Signals architecture. Components should not depend on Zone.js change-detection behavior. The production build gate verifies dead-code elimination, minification, and reactive Signals bindings.

### 5.3 Local Verification

Once implementation exists, local verification should use the Cloudflare Workers/Miniflare emulator through the API runtime, typically `http://localhost:8787`.

---

## 6. Code Quality Expectations

- No explicit `any` types are permitted. If a rare escape hatch is approved, document the reason in the code review, not as a code comment.
- Business logic must live in the domain layer, not in database constraints or UI components.
- API contracts must match `docs/specs/api-endpoints.md`.
- Schema changes must match `docs/specs/database-schema.md`.
- Authentication changes must match `docs/specs/auth-sessions.md`.
- Betting math, exposure caps, and ADL must match `docs/specs/betting-engine.md`.
- Currency and withdrawal mechanics must match `docs/specs/monetization-engine.md`.

---

## 7. Feature Branch Checklist

Before a feature branch is considered ready for review, confirm:

- [ ] The relevant spec was updated before implementation code.
- [ ] `docs/SPECS.md` still matches the domain specs.
- [ ] `docs/CONSTITUTION.md` was updated if an architectural boundary changed.
- [ ] API contracts, schema, auth, betting, or monetization docs were updated when applicable.
- [ ] Target validation gates are either passing or clearly marked as unavailable because the repo is still spec-only.
- [ ] No documentation file contains contradictory source-of-truth claims.
