# ACBU Smart Contract Specification

This document describes the on-chain Soroban (Stellar) smart contracts for ACBU, including interfaces, data structures, events, and invariants.

---

## Overview

The ACBU stablecoin system consists of 7 core Soroban contracts written in Rust:

| # | Contract | Purpose |
|---|----------|---------|
| 1 | `acbu_minting` | Mint ACBU from USDC/fiat with fees |
| 2 | `acbu_burning` | Burn ACBU, trigger fiat redemption |
| 3 | `acbu_oracle` | Multi-validator rate feed, median rates |
| 4 | `acbu_reserve_tracker` | Reserve balances per currency |
| 5 | `acbu_savings_vault` | Lock ACBU, enforce terms, yield accrual |
| 6 | `acbu_lending_pool` | Deposits, withdraws, borrowing logic |
| 7 | `acbu_escrow` | Hold ACBU; release/refund on conditions |

Shared code lives in `shared/` crate.

---

## Shared Types and Constants

**Location:** `contracts/shared/src/lib.rs`

### Constants

| Constant | Value | Purpose |
|----------|-------|---------|
| `BASIS_POINTS` | `10_000` | Fee and ratio calculations (bps) |
| `DECIMALS` | `7` | ACBU token decimals (7 for small unit precision) |
| `MIN_MINT_AMOUNT` | `10_000_000` | Minimum mint: 0.01 ACBU (in stroops) |
| `MAX_MINT_AMOUNT` | `1_000_000_000_000_000` | Maximum mint: ~1M ACBU |
| `MIN_BURN_AMOUNT` | `1_000_000` | Minimum burn: 0.0001 ACBU |
| `UPDATE_INTERVAL_SECONDS` | `3600` | Oracle update interval (1 hour) |
| `EMERGENCY_THRESHOLD_BPS` | `8000` | Reserve ratio emergency threshold (80%) |
| `OUTLIER_THRESHOLD_BPS` | `500` | Outlier detection threshold (5%) |

### Shared Types

```rust
// Currency code (ISO 3-letter)
pub type CurrencyCode = Symbol;

// Rate data for a currency
pub struct RateData {
    pub rate: i128,    // Scaled by DECIMALS
    pub timestamp: u64,
    pub source: Symbol,
}

// Reserve data for a currency
pub struct ReserveData {
    pub amount: i128,
    pub value_usd: i128,
}

// Account details for fiat redemption
pub struct AccountDetails {
    pub account_type: Symbol,     // "bank" or "mobile_money"
    pub account_number: Bytes,
    pub bank_code: Bytes,
    pub account_name: Bytes,
}

// Contract error enum
pub enum ContractError {
    Unauthorized,
    Paused,
    InvalidAmount,
    InsufficientReserves,
    InvalidCurrency,
    RateStale,
    Outlier,
}
```

### Shared Helpers

- `calculate_fee(amount: i128, rate_bps: i128) -> i128` – Fee = amount * rate / BASIS_POINTS
- `calculate_amount_after_fee(amount: i128, fee_bps: i128) -> i128` – Net amount after fee
- `median(values: Vec<i128>) -> i128` – Median of sorted values (for oracle)
- `calculate_deviation(from: i128, to: i128) -> i128` – Deviation in basis points

---

## 1. acbu_minting

**Purpose:** Convert USDC or verified fiat deposits to ACBU tokens.

### Data Keys

| Key | Type | Description |
|-----|------|-------------|
| `admin` | Address | Contract admin for upgrades/pause |
| `oracle` | Address | Oracle contract reference |
| `reserve_tracker` | Address | Reserve tracker contract |
| `acbu_token` | Address | ACBU token contract |
| `usdc_token` | Address | USDC token contract |
| `fee_rate` | i128 | Fee in basis points |
| `paused` | bool | Circuit breaker flag |
| `min_mint_amount` | i128 | Minimum mint threshold |
| `max_mint_amount` | i128 | Maximum mint threshold |

### Events

```rust
pub struct MintEvent {
    pub user: Address,
    pub usdc_amount: i128,
    pub acbu_amount: i128,
    pub fee: i128,
    pub rate: i128,
    pub transaction_id: Symbol,
    pub timestamp: u64,
}
```

### Functions

| Function | Signature | Auth Required | Description |
|----------|-----------|---------------|-------------|
| `initialize` | `(admin, oracle, reserve_tracker, acbu_token, usdc_token, fee_rate_bps)` | No | One-time setup |
| `mint_from_usdc` | `(user, usdc_amount)` | User | Accepts USDC, mints ACBU |
| `mint_from_fiat` | `(user, amount, fintech_tx_id)` | User | Mints from verified fiat |

### Invariants

- `mint_from_usdc` requires valid oracle rate and sufficient reserves
- Total supply always backed by reserves (verified on-chain)
- Fees taken before mint, recorded in event
- Each `fintech_tx_id` is unique (no replay)

---

## 2. acbu_burning

**Purpose:** Burn ACBU tokens in exchange for fiat redemption.

### Data Keys

| Key | Type | Description |
|-----|------|-------------|
| `admin` | Address | Contract admin |
| `oracle` | Address | Oracle contract |
| `reserve_tracker` | Address | Reserve tracker |
| `acbu_token` | Address | ACBU token |
| `withdrawal_processor` | Address | Backend processor address |
| `fee_rate` | i128 | Fee in basis points |
| `paused` | bool | Circuit breaker |
| `min_burn_amount` | i128 | Minimum burn threshold |

### Events

```rust
pub struct BurnEvent {
    pub user: Address,
    pub acbu_amount: i128,
    pub local_amount: i128,
    pub currency: CurrencyCode,
    pub fee: i128,
    pub rate: i128,
    pub account_details: AccountDetails,
    pub recipient_accounts: Vec<Address>,
    pub timestamp: u64,
}
```

### Functions

| Function | Signature | Auth Required | Description |
|----------|-----------|---------------|-------------|
| `initialize` | `(admin, oracle, reserve_tracker, acbu_token, withdrawal_processor, fee_rate_bps)` | No | Setup |
| `burn` | `(user, acbu_amount, account_details)` | User | Burn ACBU for fiat redemption |
| `burn_for_basket` | `(user, acbu_amount, account_details, recipient_accounts)` | User | Burn distributed to multiple recipients |

### Invariants

- Burn amount meets minimum threshold
- Oracle rate validated (not stale, within bounds)
- Reserve has sufficient funds for currency
- Fees calculated correctly, dust handled

---

## 3. acbu_oracle

**Purpose:** Aggregate rates from multiple validators, provide median rates.

### Data Keys

| Key | Type | Description |
|-----|------|-------------|
| `admin` | Address | Admin for validator management |
| `validators` | Vec<Address> | Authorized validators |
| `min_signatures` | u32 | Quorum threshold for updates |
| `rates` | Map<CurrencyCode, RateData> | Current rates |
| `last_update` | u64 | Last update timestamp |
| `update_interval` | u64 | Min seconds between updates |
| `basket_weights` | Map<CurrencyCode, i128> | Currency weights in basket |

### Events

```rust
pub struct RateUpdateEvent {
    pub currency: CurrencyCode,
    pub old_rate: i128,
    pub new_rate: i128,
    pub validators: Vec<Address>,
    pub timestamp: u64,
}
```

### Functions

| Function | Signature | Auth Required | Description |
|----------|-----------|---------------|-------------|
| `initialize` | `(admin, validators, min_signatures, currencies, basket_weights)` | No | Setup |
| `submit_rates` | `(validator, rates)` | Validator | Submit rates for multiple currencies |
| `get_rate` | `(currency) -> RateData` | No | Get current rate |
| `get_rates` | `() -> Map<CurrencyCode, RateData>` | No | Get all rates |

### Invariants

- Only authorized validators can submit rates
- Rates rejected if stale (> update_interval)
- Outlier rates rejected (> OUTLIER_THRESHOLD_BPS deviation)
- Median computed only with `len(validators) >= min_signatures`

---

## 4. acbu_reserve_tracker

**Purpose:** Track reserve balances per currency, verify collateralization.

### Data Keys

| Key | Type | Description |
|-----|------|-------------|
| `admin` | Address | Admin for reserve updates |
| `acbu_token` | Address | ACBU token contract |
| `reserves` | Map<CurrencyCode, ReserveData> | Reserve per currency |
| `min_ratio` | i128 | Minimum collateralization (bps) |
| `target_ratio` | i128 | Target ratio (bps) |

### Events

```rust
pub struct ReserveUpdateEvent {
    pub currency: CurrencyCode,
    pub amount: i128,
    pub value_usd: i128,
    pub timestamp: u64,
}
```

### Functions

| Function | Signature | Auth Required | Description |
|----------|-----------|---------------|-------------|
| `initialize` | `(admin, acbu_token, min_ratio_bps, target_ratio_bps)` | No | Setup |
| `update_reserve` | `(currency, amount, value_usd)` | Admin | Update reserve amount |
| `get_reserve_ratio` | `() -> i128` | No | Compute total ratio |
| `get_reserves` | `() -> Map<CurrencyCode, ReserveData>` | No | Get all reserves |
| `verify_reserves` | `(required_ratio_bps) -> bool` | No | Check collateralization |

### Invariants

- `verify_reserves` reads actual minted supply from token contract
- Reserve amounts always >= 0
- Admin-only updates

---

## 5. acbu_savings_vault

**Purpose:** Lock ACBU tokens for fixed terms with yield accrual.

### Data Keys

| Key | Type | Description |
|-----|------|-------------|
| `admin` | Address | Admin |
| `acbu_token` | Address | ACBU token |
| `fee_rate` | i128 | Fee on deposit/withdraw |
| `positions` | Map<(Address, u64), Position> | User positions by account+term |
| `term_seconds` | Map<Address, u64> | Active term for each user |
| `paused` | bool | Circuit breaker |

### Types

```rust
pub struct Position {
    pub amount: i128,
    pub deposit_timestamp: u64,
    pub yield_accrued: i128,
    pub locked: bool,
}
```

### Events

```rust
pub struct DepositEvent {
    pub user: Address,
    pub amount: i128,
    pub term_seconds: u64,
    pub timestamp: u64,
}

pub struct WithdrawEvent {
    pub user: Address,
    pub amount: i128,
    pub yield_amount: i128,
    pub fee: i128,
    pub timestamp: u64,
}
```

### Functions

| Function | Signature | Auth Required | Description |
|----------|-----------|---------------|-------------|
| `initialize` | `(admin, acbu_token, fee_rate_bps)` | No | Setup |
| `deposit` | `(user, amount, term_seconds)` | User | Lock ACBU for term |
| `withdraw` | `(user, amount)` | User | Withdraw after term |
| `get_position` | `(user) -> Position` | No | Get user position |

### Invariants

- Term enforced: early withdraw fails or pays penalty
- Yield computed based on time and rate
- User must authorize their own deposit/withdraw

---

## 6. acbu_lending_pool

**Purpose:** Liquidity pool for lending deposits and borrows.

### Data Keys

| Key | Type | Description |
|-----|------|-------------|
| `admin` | Address | Admin |
| `acbu_token` | Address | ACBU token |
| `fee_rate` | i128 | Fee on operations |
| `lender_balances` | Map<Address, i128> | Deposited amounts |
| `loans` | Map<u64, Loan> | Active loans |
| `paused` | bool | Circuit breaker |

### Types

```rust
pub struct Loan {
    pub borrower: Address,
    pub amount: i128,
    pub interest_rate: i128,
    pub start_timestamp: u64,
    pub due_timestamp: u64,
    pub repaid: i128,
}
```

### Events

```rust
pub struct DepositEvent {
    pub lender: Address,
    pub amount: i128,
    pub timestamp: u64,
}

pub struct WithdrawEvent {
    pub lender: Address,
    pub amount: i128,
    pub fee: i128,
    pub timestamp: u64,
}

pub struct LoanCreatedEvent {
    pub loan_id: u64,
    pub borrower: Address,
    pub amount: i128,
    pub interest_rate: i128,
    pub timestamp: u64,
}

pub struct RepaymentEvent {
    pub loan_id: u64,
    pub amount: i128,
    pub timestamp: u64,
}
```

### Functions

| Function | Signature | Auth Required | Description |
|----------|-----------|---------------|-------------|
| `initialize` | `(admin, acbu_token, fee_rate_bps)` | No | Setup |
| `deposit` | `(lender, amount)` | Lender | Add liquidity |
| `withdraw` | `(lender, amount)` | Lender | Remove liquidity |
| `borrow` | `(borrower, amount, interest_rate, duration_seconds)` | Borrower | Take loan |
| `repay` | `(borrower, loan_id, amount)` | Borrower | Repay loan |

### Invariants

- Lender must authorize their own deposit/withdraw
- Fees applied to interest calculations
- Loan lifecycle properly tracked

---

## 7. acbu_escrow

**Purpose:** Hold ACBU for conditional release, used by merchant gateways.

### Data Keys

| Key | Type | Description |
|-----|------|-------------|
| `admin` | Address | Admin |
| `acbu_token` | Address | ACBU token |
| `escrows` | Map<(Address, u64), Escrow>` | Escrow by payer+sequence |
| `paused` | bool | Circuit breaker |

### Types

```rust
pub struct Escrow {
    pub payer: Address,
    pub payee: Address,
    pub amount: i128,
    pub status: Symbol,  // "created", "funded", "released", "refunded", "disputed"
    pub created_timestamp: u64,
    pub release_timestamp: u64,  // Optional timeout
}
```

### Events

```rust
pub struct EscrowCreatedEvent {
    pub escrow_id: (Address, u64),
    pub payer: Address,
    pub payee: Address,
    pub amount: i128,
    pub timestamp: u64,
}

pub struct EscrowReleasedEvent {
    pub escrow_id: (Address, u64),
    pub payee: Address,
    pub amount: i128,
    pub timestamp: u64,
}
```

### Functions

| Function | Signature | Auth Required | Description |
|----------|-----------|---------------|-------------|
| `initialize` | `(admin, acbu_token)` | No | Setup |
| `create` | `(payer, payee, amount, release_timestamp?)` | Payer | Create escrow |
| `fund` | `(payer, escrow_id)` | Payer | Deposit ACBU |
| `release` | `(payer, escrow_id)` | Payer/Admin | Release to payee |
| `refund` | `(payer, escrow_id)` | Payer/Admin | Refund to payer |
| `dispute` | `(payer, escrow_id)` | Payer | Flag for dispute |

### Invariants

- Escrow ID is unique per payer (payer + sequence)
- Only payer or admin can release
- Funds locked until release/refund/dispute resolution

---

## Security Model

### Authorization Patterns

- All user-modifying functions require `user.require_auth()`
- Admin functions check `caller == admin`
- Validator functions check membership in `validators` set
- Pause flag checked on all mutating entrypoints

### Reentrancy Safety

- Checks-effects-interactions pattern followed
- Token transfers happen after state updates
- No callbacks into untrusted contracts

### Upgrade Path

- All contracts store `admin` address
- Future upgrade requires admin migration flow
- Versioned storage keys for forwards compatibility

---

## Known Issues (from contracts.md)

### Critical
- C-001: Escrow `release` missing authorization
- C-002: `mint_from_fiat` lacks off-chain settlement proof
- C-003: Escrow ID key collision risk
- C-004: `verify_reserves` uses wrong supply source
- C-005: Savings vault missing auth checks
- C-006: Lending pool missing auth checks

### High
- C-007: Savings term not enforced
- C-008: `mint_from_fiat` missing max mint bound
- C-009: Burn basket integer division dust loss
- C-010: Escrow create lacks duplicate guard
- C-011: Minting loads oracle but doesn't call

### Medium
- C-013: Token WASM import uses zero SHA256
- C-014: Double unwrap in escrow storage
- C-017: Oracle outlier detection is no-op
- C-020: `median` allocates via `to_vec()` in no_std

---

## Testing Contracts

### Running Tests

```bash
# Unit tests for all contracts
cargo test --workspace

# Specific contract tests
cargo test -p acbu_oracle
cargo test -p acbu_minting
cargo test -p acbu_burning
```

### Test Coverage Goals

- All public functions have unit tests
- Events emitted correctly
- Authorization enforced
- Edge cases (zero, overflow, stale rates) covered

---

## Build and Deploy

```bash
# Build all contracts
cargo build --target wasm32-unknown-unknown --release

# Output: target/wasm32-unknown-unknown/release/acbu_*.wasm
```

### WASM Import Hashes

Token contract WASM hash must be pinned in production:
- `acbu_minting`: Pin real hash for `soroban_token_contract.wasm`
- `acbu_burning`: Pin real hash
- `acbu_reserve_tracker`: Pin real hash

---

## Related Documents

- [API Specification](../TECHNICAL/API_SPECIFICATION.MD) – REST API endpoints
- [API and Contracts Reference](../PROJECT/API_AND_CONTRACTS_REFERENCE.MD) – Routes + contracts table
- [Project Structure](../PROJECT/PROJECT_STRUCTURE.MD) – Full repo layout
- [Contract Issues](../issues/contracts.md) – Detailed issue catalog