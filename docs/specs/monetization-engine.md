# Monetization & Internal Liquidity Engine — Edgy Stonks

This document specifies the closed-loop economic tokenomics, virtual currency pegging, withdrawal mechanics, and zero-passive-liability design.

---

## 1. Core Currency: mETH

The application uses an internal utility token named **mETH**.

- **Peg Ratio:** `1 mETH = 0.001 ETH`.
- **Backing Model:** mETH minting is paired with delta-neutral liquidity management so the platform does not absorb directional ETH treasury risk.
- **Value Dynamics:** mETH purchasing and liquidation power tracks the global market price of Ethereum.

---

## 2. Dual-Ledger Model

The platform maintains two ledgers:

| Ledger | Purpose | Withdrawable |
| --- | --- | --- |
| `virtual_balance_cents` | Internal non-withdrawable casino currency. | No |
| `meth_balance_cents` | mETH operations and settlement accounting. | Yes |

Integer accounting units are authoritative for balances and payouts. The `_cents` suffix means integer hundredths of the ledger unit, not fiat dollars.

Users can withdraw only mETH through the Base L2 settlement flow.

---

## 3. Closed-Loop Peer-to-Peer Betting Mechanics

The platform operates as a synthetic pool isolated from traditional equity markets.

- Users place `LONG` or `SHORT` leveraged bets on equities such as `NVDA` and `TSLA`.
- Users are not buying underlying shares.
- Users are wagering against the aggregate liquidity pool of other users inside the application.
- The betting engine calculates payouts and liquidations from official market data, but payouts are drawn from the internal closed-loop mETH and virtual ledgers.

---

## 4. Revenue Architecture: The Peage Model

The platform monetizes through structural friction during liquidity exits rather than out-performing the user.

1. **Withdrawal Spread Fee:** A fixed percentage fee, for example `2.5%`, is applied to outbound redemption of mETH back into external Ether.
2. **Network Dust Absorption:** Residual fractions below the minimum transaction threshold remain within the platform's core operational wallet to offset transaction-level settlement costs.

### 4.1 Precision and Floating-Point Rounding Strategy

The platform avoids heavy arbitrary-precision decimal libraries inside Cloudflare Workers.

When real-time market prices from `assets` or `bets` interact with integer balance columns during buy, sell, liquidation, or withdrawal triggers, the platform applies a strict floor truncation strategy using JavaScript's native `Math.floor()`.

Fractional losses are dropped at the domain-engine boundary. The core ledger preserves integer consistency.

---

## 5. Blockchain Infrastructure & Settlement Layer

The platform uses Base L2 for tokenized value settlement.

- **Settlement Network:** Base, selected for sub-cent gas fees, immediate block finality, and deep corporate ecosystem liquidity.
- **Backend Runtime Library:** `viem`.
- **RPC Gateway:** A configurable RPC URL secret, for example `BASE_RPC_URL`.
- **Key Custody & Signing:** The central vault private key is injected at runtime through Cloudflare Worker Secrets, for example `BASE_VAULT_PRIVATE_KEY`. It is never exposed in source code or frontend bundles.
- **Transaction Signing:** Transaction signing happens inside isolated V8 memory space using Web Standard APIs through `viem`. No heavy full-node execution is required inside Cloudflare Workers.

The RPC provider is intentionally abstracted. Documentation must not hardcode or authorize a single third-party provider.

---

## 6. Asset Pricing, Deposit, and Withdrawal Settlement Mechanics

- Internal virtual currency values, asset exposures, and user balances are accounted for through the dual-ledger model.
- mETH is the only withdrawable ledger.
- Deposits fund mETH operations by sending physical ETH to the platform's Base L2 liquidity address.
- Credit cards, bank transfers, and traditional payment gateways are unsupported.
- Withdrawals audit the internal mETH ledger, convert approved mETH back to physical ETH at `1000 mETH = 1 ETH`, subtract the configured withdrawal fee, and send the result to the user's verified Base L2 wallet address.
- Outbound settlements are dispatched only to verified user-controlled Base L2 wallet addresses. Users are responsible for providing accurate and compatible wallet addresses.

---

## 7. System Intermittency & Limitation of Liability

The application is an experimental Proof of Concept deployed across distributed systems and public blockchain networks.

- The platform, developers, and operators hold no liability for temporary financial invisibility, synchronization delays, transactional stalls, or inability to deposit or withdraw assets caused by intermittent system failures.
- Service availability depends on third-party infrastructure, including Cloudflare Worker routing, D1 availability, RPC gateway availability, and Base network consensus.
- API rate limiting, network partitions, and provider outages are handled as asynchronous technical anomalies.
- The platform guarantees no real-time availability SLAs. Users assume operational risks related to transient network latency or unexpected service downtime.
