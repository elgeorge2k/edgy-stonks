# Specification: Betting Engine & Multiplier Mathematics

## 1. Context & Core Philosophy

The betting engine evaluates active player positions against real-world market deltas. To simulate a high-dopamine casino experience, the engine applies a leverage multiplier to short-term stock movements.

The domain betting engine owns betting math, exposure caps, dynamic payout scaling, and ADL behavior. D1 stores resulting ledger state and must not dictate business rules.

## 2. Mathematical Model

### 2.1 Asset Price Variance

When an evaluation is triggered through the consumer-driven hydration flow, the engine calculates the percentage change of the asset since the position opened:

```text
price_variance = (current_price - buy_price) / buy_price
```

### 2.2 Nominal Betting Multiplier

The engine uses a fixed leverage coefficient of `50`. The raw mathematical multiplier before pool-liquidity constraints is:

```text
nominal_multiplier = 1 + (price_variance * 50)
```

Position return logic:

- **Long position:** `final_return = bet_amount_cents * nominal_multiplier`
- **Short position:** `final_return = bet_amount_cents * (2 - nominal_multiplier)`

### 2.3 Liquidation Threshold

A position is liquidated when the asset moves against the player past the liquidation threshold.

For `L = 50`, if the real stock moves `2%` or more in the wrong direction, the position value is wiped out.

## 3. Execution Mechanics

### 3.1 Step 1 — Intention

`POST /api/v1/bets` performs the following domain checks before persistence:

1. Validates the selected ledger: `VIRTUAL` or `METH`.
2. Validates the selected ticker and bet direction.
3. Checks global exposure caps for ticker and direction.
4. Calculates the current payout ratio string.
5. Deducts `collateral_amount_cents` from the selected user ledger.
6. Creates a row in `bets` with `status = 'NEW'`.

The response includes:

- `bet_id`
- `payout_ratio_string`
- exposure and liquidity metadata required by the frontend before final confirmation.

### 3.2 Step 2 — Lock & Execution

`PUT /api/v1/bets/:id` validates that the transaction has not already been processed. It then:

1. Locks the live asset price from `assets`.
2. Updates the `bets` row to `status = 'COMPLETED'`.
3. Populates `buy_price`.
4. Inserts the active position into `portfolios`.

The `portfolios` table remains untouched until Step 2 completes.

## 4. Resolution Workflow

1. The user requests to cash out or close an active position through `POST /api/v1/bets/:id/cashout` followed by `PUT /api/v1/bets/:id/cashout`.
2. The Worker reads the latest cached `current_price` from `assets`.
3. The domain engine computes asset variance, final multiplier, payout, fees, and any ADL adjustment.
4. The atomic database transaction:
   - Credits `payout_amount_cents` back to the user's corresponding profile balance in `users`.
   - Updates `settlement_price`, `final_multiplier`, `payout_amount_cents`, and `resolved_at` on the `bets` row.
   - Sets `status = 'COMPLETED'`.
   - Sets `resolution_status = 'CASHED_OUT'`.
   - Deletes the active tracking position from `portfolios`.
5. Floating-point operations enforce safe checks to prevent division-by-zero errors from downstream pricing anomalies.

## 5. Pool Solvency & Risk Mitigation

The platform operates as a risk-neutral intermediary. The domain engine enforces real-time mathematical constraints during position entry and resolution.

### 5.1 Global Exposure Caps

The engine restricts the maximum cumulative collateral permitted for any ticker and direction. If `POST /api/v1/bets` violates this boundary, the engine aborts execution before persistence.

### 5.2 Dynamic Payout Scaling

Payout multipliers are bounded by the active ratio of opposing positions inside the global pool ledger.

The engine computes the liquidity balancing ratio during position entry and exposes it to the UI as `payout_ratio_string`.

Formula logic:

```text
final_multiplier = nominal_multiplier * (opposing_volume / target_volume)
```

Examples:

- **Saturated direction:** `Pays 1:40`
  - The crowded side receives compressed returns.
  - Final multiplier is reduced by the opposing-volume ratio.

- **Premium contrarian direction:** `Pays 40:1`
  - The contrarian side receives amplified returns.
  - Amplification is funded by liquidations and reduced payouts from the saturated opposing side.

### 5.3 Zero-House-Risk Enforcer

If directional winnings mathematically exceed available pool collateral, the domain engine triggers ADL and caps payouts at the absolute liquidity limit of the pool. The company treasury is never exposed to user payout liabilities.

D1 records the capped result; D1 does not calculate or dictate the cap.
