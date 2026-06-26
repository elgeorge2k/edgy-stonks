# Domain Model & Ubiquitous Language

This document defines the core Domain-Driven Design (DDD) objects for the EdgyStonks platform. It serves as the single source of truth for the ubiquitous language used in the codebase, ensuring business invariants are strictly enforced in memory before any database interaction occurs.

## 1. Value Objects (VO)
*Immutable objects that describe a characteristic or attribute but have no conceptual identity. They self-validate upon instantiation.*

* **`MoneyMethCents`**: Represents real money within the system. 
  * *Constraint*: Must be an integer ($1 \text{ cent} = 10,000 \text{ Gwei}$). Cannot be negative.
* **`MoneyVirtualCents`**: Represents leveraged synthetic exposure (used for virtual shares).
* **`PayoutRatio`**: Represents the dynamic liquidity balancing multiplier (e.g., "40:1" or "1:40").
  * *Constraint*: Derived from the ratio of opposing pool volume vs target pool volume.
* **`Multiplier`**: The fixed leverage coefficient (Always `50x`).
* **`WalletAddress`**: A cryptographic L2 Base network address.
  * *Constraint*: Must pass regex validation for a standard EVM `0x...` hex string.
* **`AssetPrice`**: The raw fractional price of a stock/crypto ticker fetched from the oracle.

## 2. Entities
*Mutable objects that have a distinct identity that runs through time and different states.*

* **`User`**: The core identity holder, tied to a federated OAuth `provider_user_id`.
* **`Wallet`**: Manages the user's real `MoneyMethCents` balance.
* **`Bet` / `Position`**: Tracks a specific wager on a `Ticker`. Transitions through states (`NEW`, `PENDING`, `ACTIVE`, `RESOLVED`).
* **`CryptoTransaction`**: Represents a physical L2 blockchain movement (`Deposit` or `Withdrawal`). Tracks the `TransactionHash`.
* **`Ticker`**: The curated market asset (e.g., `NVDA`, `BTC`) acting as the underlying reference for positions.

## 3. Aggregates (Aggregate Roots)
*Clusters of domain objects that can be treated as a single unit for data changes. The Aggregate Root is the only class outside objects are allowed to hold references to, ensuring internal consistency.*

### 3.1 LiquidityPool (The Market Guardian)
**Purpose:** Protects the platform's solvency by enforcing the Zero-House-Risk mathematical invariants for a specific `Ticker`. It acts as the absolute boundary that decides if a new wager can enter the system and at what dynamic multiplier.

* **Internal State:**
  * `ticker`: The `Ticker` entity this pool represents.
  * `totalLongVolume`: `MoneyMethCents` representing total collateral locked in UP positions.
  * `totalShortVolume`: `MoneyMethCents` representing total collateral locked in DOWN positions.
  * `exposureCap`: `MoneyMethCents` representing the absolute maximum allowed liquidity imbalance for this asset (ADL limit).

* **Domain Behaviors & Invariants:**
  * `evaluatePayoutRatio(targetDirection)`: Computes the dynamic balancing multiplier based on the current volume asymmetry. 
    * *Rule:* Applies the linear logic `nominal_multiplier * (opposing_volume / target_volume)` and returns a strictly formatted `PayoutRatio` VO (e.g., "1:40" or "40:1").
  * `canAcceptExposure(betAmountCents, targetDirection)`: Validates if the incoming wager would push the pool beyond its `exposureCap`.
  * `registerBet(user, betAmountCents, direction)`: The core mutation method. 
    * *Execution:* Checks `canAcceptExposure`. If invalid, immediately throws a `LiquidityPoolExposureException` (preventing database insertion). If valid, it updates the respective volume state, instantiates a valid `Bet` entity in a `NEW` state, and locks in the computed `PayoutRatio` for that specific bet.

### 3.2 UserPortfolio (The Wallet Guardian)
**Purpose:** Enforces user-level financial invariants. It ensures that money cannot be created out of thin air, preventing negative balances, unauthorized double-spending, and managing the strict lifecycle of a user's wallet collateral.

* **Internal State:**
  * `user`: The `User` entity owning the portfolio.
  * `wallet`: The `Wallet` entity tracking the available `MoneyMethCents` balance.
  * `activeBets`: A collection of unresolved `Bet` entities belonging to the user.

* **Domain Behaviors & Invariants:**
  * `allocateCollateral(betAmountCents)`: The authorization gateway for placing a bet. 
    * *Rule:* Validates if `wallet.balance >= betAmountCents`. If false, immediately throws an `InsufficientFundsException`. If true, it atomically deducts the `betAmountCents` from the wallet's available balance and authorizes the creation of the `Bet` entity.
  * `resolvePosition(bet, payoutAmountCents)`: Handles the resolution of a finished position. 
    * *Rule:* Adds the mathematically verified `payoutAmountCents` (which may be zero, partial, or multiplied) back into the `wallet.balance`.
  * `lockForWithdrawal(withdrawAmountCents)`: 
    * *Rule:* Deducts funds from the `wallet` *before* the blockchain transaction is broadcasted to Base L2 via `viem`, preventing a double-spend race condition.
  * `creditValidatedDeposit(depositAmountCents, transactionHash)`: 
    * *Rule:* Increases the `wallet.balance` only if the `transactionHash` is unique and mathematically verified by the L2 RPC.