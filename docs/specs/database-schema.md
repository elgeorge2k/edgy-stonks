# Specification: Database Schema & Relational Design

## 1. Context & Engine Constraints

This specification targets Cloudflare D1, a SQLite-compatible edge database. All tables use strict data typing. Multi-tenant partitioning relies on `user_id` text fields derived from stateless authentication.

D1 is the ledger and cache store. Business rules such as betting math, exposure caps, dynamic payout scaling, and ADL remain in the domain betting engine.

## 2. Entity Relationship Schema (DDL)

### 2.1 Users Table

Stores core balances and registration timestamps. Identity is federated through OAuth2 providers such as Google and Microsoft. The platform does not manage credentials or custom usernames.

`virtual_balance_cents` tracks internal non-withdrawable casino currency. `meth_balance_cents` tracks mETH operations and is the withdrawable settlement ledger.

```sql
CREATE TABLE users (
    id TEXT PRIMARY KEY,
    provider_name TEXT NOT NULL,
    provider_user_id TEXT NOT NULL,
    display_name TEXT NOT NULL,
    avatar_url TEXT,
    virtual_balance_cents INTEGER NOT NULL DEFAULT 1000000,
    meth_balance_cents INTEGER NOT NULL DEFAULT 0,
    verified_wallet_address TEXT UNIQUE,
    created_at INTEGER NOT NULL DEFAULT (strftime('%s', 'now')),
    updated_at INTEGER NOT NULL DEFAULT (strftime('%s', 'now'))
);

CREATE UNIQUE INDEX idx_provider_identity ON users(provider_name, provider_user_id);
```

### 2.2 Assets Table

Stores the current market snapshot of tracked tickers. Market synchronization hydrates this table through request-time stale-first cache behavior and asynchronous `ctx.waitUntil()` hydration.

```sql
CREATE TABLE assets (
    ticker TEXT PRIMARY KEY,
    company_name TEXT NOT NULL,
    current_price REAL NOT NULL,
    last_updated_at INTEGER NOT NULL,
    is_active INTEGER NOT NULL DEFAULT 1
);
```

### 2.3 Asset History Table

Stores historical daily aggregates to render technical-analysis charts.

```sql
CREATE TABLE asset_history (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    ticker TEXT NOT NULL,
    historical_date TEXT NOT NULL,
    open_price REAL NOT NULL,
    high_price REAL NOT NULL,
    low_price REAL NOT NULL,
    close_price REAL NOT NULL,
    FOREIGN KEY (ticker) REFERENCES assets(ticker) ON DELETE CASCADE
);

CREATE UNIQUE INDEX idx_ticker_date ON asset_history(ticker, historical_date);
```

### 2.4 User Portfolio Table

Tracks active bets on stocks the user currently owns, including execution price and bet size.

```sql
CREATE TABLE portfolios (
    id TEXT PRIMARY KEY,
    user_id TEXT NOT NULL,
    ticker TEXT NOT NULL,
    shares_quantity REAL NOT NULL,
    average_buy_price REAL NOT NULL,
    opened_at INTEGER NOT NULL DEFAULT (strftime('%s', 'now')),
    bet_type TEXT NOT NULL CHECK (bet_type IN ('LONG', 'SHORT')),
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE RESTRICT,
    FOREIGN KEY (ticker) REFERENCES assets(ticker) ON DELETE RESTRICT
);
```

### 2.5 Bets Table

Centralized lifecycle ledger tracking positions from creation to resolution.

```sql
CREATE TABLE bets (
    id TEXT PRIMARY KEY,
    user_id TEXT NOT NULL,
    ticker TEXT NOT NULL,
    bet_type TEXT NOT NULL CHECK (bet_type IN ('LONG', 'SHORT')),
    balance_type TEXT NOT NULL CHECK (balance_type IN ('VIRTUAL', 'METH')),
    collateral_amount_cents INTEGER NOT NULL,
    buy_price REAL NOT NULL,
    settlement_price REAL,
    final_multiplier REAL,
    payout_amount_cents INTEGER,
    status TEXT NOT NULL CHECK (status IN ('NEW', 'PENDING', 'COMPLETED', 'FAILED')),
    resolution_status TEXT CHECK (resolution_status IN ('CASHED_OUT', 'LIQUIDATED')),
    opened_at INTEGER NOT NULL,
    resolved_at INTEGER,
    attempts INTEGER NOT NULL DEFAULT 0,
    error_message TEXT,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE RESTRICT,
    FOREIGN KEY (ticker) REFERENCES assets(ticker) ON DELETE RESTRICT
);

CREATE INDEX idx_bets_status ON bets(status);
CREATE INDEX idx_bets_user_status ON bets(user_id, status);
```

### 2.6 Revoked Tokens Table

Stores token hashes that have been explicitly invalidated through logout or security revocation before natural expiration.

```sql
CREATE TABLE revoked_tokens (
    token_hash TEXT PRIMARY KEY,
    user_id TEXT NOT NULL,
    expires_at INTEGER NOT NULL,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

CREATE INDEX idx_revoked_tokens_user_expires ON revoked_tokens(user_id, expires_at);
```

### 2.7 Crypto Transactions Table

On-chain Base L2 settlement ledger for deposits and withdrawals.

```sql
CREATE TABLE crypto_transactions (
    id TEXT PRIMARY KEY,
    user_id TEXT NOT NULL,
    transaction_type TEXT NOT NULL CHECK (transaction_type IN ('DEPOSIT', 'WITHDRAWAL')),
    meth_amount_cents INTEGER NOT NULL,
    eth_amount_real REAL NOT NULL,
    tx_hash TEXT UNIQUE,
    status TEXT NOT NULL CHECK (status IN ('NEW', 'PENDING', 'CONFIRMED', 'FAILED')),
    fee_charged_cents INTEGER NOT NULL DEFAULT 0,
    attempts INTEGER NOT NULL DEFAULT 0,
    error_message TEXT,
    created_at INTEGER NOT NULL DEFAULT (strftime('%s', 'now')),
    updated_at INTEGER NOT NULL DEFAULT (strftime('%s', 'now')),
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE RESTRICT
);

CREATE INDEX idx_crypto_transactions_status ON crypto_transactions(status);
```
