# Documentation Consolidation Plan

## Objective

Review and consolidate Edgy Stonks documentation so each file has a single responsibility and no conflicting source-of-truth claims remain. This task is documentation-only and does not include implementation work.

## Clarified Decisions

1. **Validation commands**
   - README.md and docs/CONTRIBUTING.md should describe validation/build commands as future target gates, not current runnable commands.

2. **Market synchronization**
   - Authoritative model is hybrid:
     - Angular visibility state controls request frequency.
     - Worker enforces a 5-minute stale-first cache and asynchronous D1 hydration rule.

3. **Authentication flow**
   - Authoritative flow is OAuth code flow:
     - `GET /api/v1/auth/login` redirects to the identity provider.
     - `GET /api/v1/auth/callback` exchanges the auth code and issues the application JWT.

4. **Currency and ledger model**
   - Keep dual ledgers:
     - `virtual_balance_cents` for internal virtual casino currency.
     - `meth_balance_cents` for mETH operations.
     - Users can only withdraw mETH.

5. **Revoked token cleanup**
   - No persistent background worker or cron.
   - Cleanup is request-time and bounded:
     - Run after the client response using `ctx.waitUntil()`.
     - Limit the maintenance work so it completes within the request lifecycle.
     - Use an integer-safe SQLite timestamp comparison, e.g. `DELETE FROM revoked_tokens WHERE expires_at < strftime('%s', 'now') LIMIT 50;`.

6. **Documentation structure**
   - Keep modular architecture blueprint:
     - README.md: high-level stakeholder overview.
     - CONTRIBUTING.md: developer/AI onboarding and validation workflow.
     - CONSTITUTION.md: immutable architectural boundaries.
     - SPECS.md: technical index and global infrastructure constraints.
     - Sub-specs: domain-isolated implementation details.

7. **ADL and exposure caps**
   - Domain betting engine is authoritative.
   - D1 stores resulting ledger state and does not dictate business rules.

8. **Crypto runtime**
   - `viem` is authoritative for Cloudflare Workers/Base L2 settlement because it aligns with Web Standard APIs, tree-shaking, V8 isolate startup, and Edge bundle constraints.

9. **RPC and market-data endpoints**
   - Document configurable endpoint secrets/environment variables rather than hardcoding third-party provider URLs.

## Audit Findings to Resolve

1. README.md duplicates documentation index and implementation details that belong in CONTRIBUTING.md, CONSTITUTION.md, and SPECS.md.
2. CONTRIBUTING.md mandatory reading order omits domain specs referenced by README.md and SPECS.md, including market-sync, monetization-engine, and api-endpoints.
3. CONTRIBUTING.md validation commands are currently aspirational because the repo is spec-only; commands must be marked as future target gates.
4. SPECS.md and market-sync.md conflict on market data synchronization:
   - SPECS.md describes client visibility-based 1-minute, 5-minute, and 15-minute behavior.
   - market-sync.md describes server-side 5-minute stale-first hydration.
   - Consolidate into the hybrid model.
5. SPECS.md mentions Massive.com provider details that should be reduced to configurable endpoint guidance.
6. auth-sessions.md and api-endpoints.md conflict on auth callback behavior:
   - auth-sessions.md describes IdP `id_token` exchange.
   - api-endpoints.md describes OAuth code flow.
   - Consolidate to OAuth code flow.
7. market-sync.md mentions Cloudflare KV/D1 while the stack is Cloudflare D1-only.
8. market-sync.md has naming/format issues:
   - `last_update` vs `assets.last_updated_at`.
   - malformed `ctx.waitUntil()` code block.
   - typo: `refocust`.
9. monetization-engine.md and database-schema.md need clearer dual-ledger language:
   - virtual cents are internal casino currency.
   - mETH is the withdrawable settlement ledger.
10. monetization-engine.md currently says `ethers.js (or viem)` and names Alchemy/QuickNode; update to `viem` plus configurable RPC URL secret.
11. betting-engine.md has DDD conflicts:
    - ADL and exposure caps must be domain-engine responsibilities, not database-ingestion-layer logic.
12. api-endpoints.md has terminology inconsistencies:
    - `betID` vs `bet_id`.
    - `collateral` vs `collateral_amount_cents`.
    - cashout status should use existing enum language consistently.
13. database-schema.md has minor consistency issues:
    - missing `FOREIGN KEY (ticker)` on `bets`.
    - crypto transaction section heading typo.
    - comments should clarify `meth_balance_cents` and non-withdrawable virtual balance.
14. CONTRIBUTING.md says no explicit `any` unless justified in a comment, which conflicts with the project’s no-unnecessary-comments rule; rephrase as a code exception rule without requiring comments.

## Implementation Steps

1. **Update README.md**
   - Keep it as a high-level overview.
   - Remove duplicated detailed architecture and validation content.
   - Point readers to CONTRIBUTING.md, CONSTITUTION.md, and SPECS.md.

2. **Update docs/CONTRIBUTING.md**
   - Replace the current quality-gate text with the modular architecture blueprint.
   - Add complete mandatory reading order across core docs and sub-specs.
   - Mark validation/build commands as future target gates.
   - Add a documentation-edit workflow:
     - spec alignment,
     - documentation-first updates,
     - validation gate review,
     - local verification after implementation exists.

3. **Update docs/CONSTITUTION.md**
   - Preserve immutable rules.
   - Add bounded request-time maintenance as the only permitted cleanup pattern for revoked tokens.
   - Clarify that D1 must not own business rules such as ADL or exposure caps.

4. **Update docs/SPECS.md**
   - Keep it as the technical index.
   - Align stack descriptions with modular docs.
   - Add the hybrid market-sync model.
   - Document OAuth code flow as the authoritative auth flow.
   - Document dual-ledger currency model at index level.
   - Use configurable provider endpoint guidance for Massive.com and RPC.

5. **Update domain specs**
   - `docs/specs/market-sync.md`: D1-only cache, hybrid client/server sync, corrected code block, `last_updated_at`, typo fixes.
   - `docs/specs/auth-sessions.md`: OAuth code flow, JWT claim language, request-time revoked-token cleanup.
   - `docs/specs/api-endpoints.md`: align endpoints and payloads with OAuth code flow, `bet_id`, `collateral_amount_cents`, and cashout status.
   - `docs/specs/database-schema.md`: dual-ledger comments, missing FK, typo fixes, clearer D1 constraints.
   - `docs/specs/betting-engine.md`: domain-engine-only ADL/exposure caps, DDD-aligned payout scaling, consistent API field names.
   - `docs/specs/monetization-engine.md`: dual-ledger withdrawal model, `viem`, configurable RPC URL secret, remove provider lock-in.

6. **Validate documentation**
   - Check all internal markdown links.
   - Search for remaining contradictory terms:
     - `id_token`,
     - `ethers.js`,
     - `Alchemy`,
     - `QuickNode`,
     - `KV/D1`,
     - `last_update`,
     - `betID`,
     - `database ingestion layer`,
     - `background asset hydration routine`.
   - Confirm README remains high-level and sub-specs remain domain-isolated.

## Validation Criteria

- No documentation file contradicts another authoritative file.
- README.md contains no implementation details that duplicate SPECS.md or domain specs.
- CONTRIBUTING.md accurately describes the current spec-only state and future target validation gates.
- SPECS.md is the core technical index without absorbing domain implementation details.
- Domain specs own their own implementation details without cross-file contradictions.
- All unresolved provider/runtime choices are either documented as configurable or replaced by the clarified choices above.
