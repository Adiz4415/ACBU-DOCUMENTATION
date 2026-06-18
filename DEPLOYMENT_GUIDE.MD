# ACBU Production Deployment Guide

This guide covers the full end-to-end steps to deploy ACBU to production. For the org-limits backfill procedure run after deployment, see [DEPLOYMENT_ORG_LIMITS_BACKFILL.md](./DEPLOYMENT_ORG_LIMITS_BACKFILL.md). For testnet bootstrap, see [TESTNET_CUSTODIAL_BOOTSTRAP.md](./TESTNET_CUSTODIAL_BOOTSTRAP.md).

---

## Prerequisites

- Node.js 18+, `pnpm` / `npm`
- Rust + `wasm32-unknown-unknown` target (for Soroban contracts)
- Soroban CLI (`stellar contract` commands)
- Stellar account with XLM for contract deployment fees
- Prisma Accelerate project (managed PostgreSQL)
- MongoDB Atlas cluster
- RabbitMQ instance (or managed broker)
- Access to fintech provider dashboards (Flutterwave, Paystack, MTN MoMo)

---

## 1. Provision Infrastructure

### 1.1 Database

ACBU uses **Prisma Accelerate** as the primary database. Create an Accelerate project at [prisma.io](https://www.prisma.io/data-platform/accelerate) and obtain the connection string:

```
prisma+postgres://accelerate.prisma-data.net/?api_key=<YOUR_KEY>
```

Set this as both `DATABASE_URL` and `PRISMA_ACCELERATE_URL` in `backend/.env`.

> **Important:** Prisma CLI reads only `backend/.env`. If you use `backend/.env.local` place `DATABASE_URL` and `PRISMA_ACCELERATE_URL` there too, or copy `.env.local` into `.env` before running migrations.

### 1.2 MongoDB Atlas

Create a MongoDB Atlas cluster for cache/session storage and obtain:

```
mongodb+srv://<user>:<password>@<cluster>.mongodb.net/acbu?retryWrites=true&w=majority
```

Set as `MONGODB_URI`.

### 1.3 RabbitMQ

Provision a RabbitMQ instance (e.g. CloudAMQP) and set `RABBITMQ_URL`:

```
amqp://<user>:<password>@<host>:5672
```

---

## 2. Configure Environment Variables

Copy all required variables into `backend/.env`. The full reference is in [ENV_VARS.md](./ENV_VARS.md). The minimum required set for production:

| Variable | Notes |
|----------|-------|
| `DATABASE_URL` | Prisma Accelerate URL |
| `PRISMA_ACCELERATE_URL` | Same value as `DATABASE_URL` |
| `MONGODB_URI` | MongoDB Atlas connection string |
| `RABBITMQ_URL` | RabbitMQ connection string |
| `JWT_SECRET` | Min 32 characters, randomly generated |
| `API_KEY_SALT` | Salt for API key hashing |
| `NODE_ENV` | Set to `production` |
| `STELLAR_NETWORK` | Set to `mainnet` |
| `STELLAR_HORIZON_URL` | Mainnet Horizon URL |
| `STELLAR_SECRET_KEY` | Backend signing key for contract transactions |
| `STELLAR_USE_DYNAMIC_FEES` | Recommended: `true` for mainnet |

Set fintech keys for each enabled payment provider (`FLUTTERWAVE_*`, `PAYSTACK_*`, `MTN_MOMO_*`).

Set oracle and reserve variables (`ORACLE_UPDATE_INTERVAL_HOURS`, `RESERVE_MIN_RATIO`, etc.) — defaults are safe for production but review against your risk parameters.

Set monitoring variables (`SENTRY_DSN`, `SENTRY_ENVIRONMENT`).

---

## 3. Run Database Migrations

Apply all pending Prisma migrations:

```bash
cd backend
npx prisma migrate deploy
```

Apply the API key lookup key migration specifically if upgrading an existing deployment:

```bash
npx prisma db execute --file prisma/sql/20260323_add_api_key_lookup_key.sql
```

Verify the schema is up to date:

```bash
npx prisma migrate status
```

---

## 4. Deploy Soroban Smart Contracts

### 4.1 Build contracts

```bash
cd acbu-smart-contract
cargo build --release --target wasm32-unknown-unknown
```

Or use the Soroban CLI build shorthand:

```bash
stellar contract build
```

### 4.2 Deploy to Mainnet

Deploy each contract and record the resulting contract ID:

```bash
stellar contract deploy \
  --wasm target/wasm32-unknown-unknown/release/acbu_oracle.wasm \
  --source <DEPLOYER_SECRET_KEY> \
  --network mainnet

stellar contract deploy \
  --wasm target/wasm32-unknown-unknown/release/acbu_reserve_tracker.wasm \
  --source <DEPLOYER_SECRET_KEY> \
  --network mainnet

stellar contract deploy \
  --wasm target/wasm32-unknown-unknown/release/acbu_minting.wasm \
  --source <DEPLOYER_SECRET_KEY> \
  --network mainnet

stellar contract deploy \
  --wasm target/wasm32-unknown-unknown/release/acbu_burning.wasm \
  --source <DEPLOYER_SECRET_KEY> \
  --network mainnet
```

### 4.3 Record Contract IDs

Set the returned contract IDs in `backend/.env`:

```
CONTRACT_ORACLE=<oracle_contract_id>
CONTRACT_RESERVE_TRACKER=<reserve_tracker_contract_id>
CONTRACT_MINTING=<minting_contract_id>
CONTRACT_BURNING=<burning_contract_id>
```

Or use per-network variants (`CONTRACT_ORACLE_MAINNET`, etc.) if supporting both testnet and mainnet from the same environment file.

### 4.4 Initialize Contracts

Run the initialization script to seed oracle basket weights, rates, and reserve values:

```bash
cd backend
npx ts-node scripts/initContracts.ts
```

Ensure `STELLAR_SECRET_KEY` is the **admin** key for all contracts before running this script.

---

## 5. Seed Oracle and Reserve Data

Before the first mint can succeed, the oracle and reserve tracker must have valid state.

### 5.1 Oracle — basket currencies and rates

For each basket currency (e.g. NGN, KES, GHS, RWF), submit an initial rate:

```bash
stellar contract invoke \
  --id $CONTRACT_ORACLE \
  --source <ADMIN_SECRET_KEY> \
  --network mainnet \
  -- submit_rate \
  --currency '["NGN"]' \
  --rate <rate_value_7_decimal_fixed_point>
```

Also call `set_s_token_address` for each currency with its deployed Stellar Asset Contract (SAC) ID.

### 5.2 Reserve tracker — initial reserve values

For each basket currency, seed a non-zero `value_usd` so `is_reserve_sufficient` passes:

```bash
stellar contract invoke \
  --id $CONTRACT_RESERVE_TRACKER \
  --source <ADMIN_SECRET_KEY> \
  --network mainnet \
  -- update_reserve \
  --currency '["NGN"]' \
  --value_usd <value_in_7_decimal_fixed_point>
```

`value_usd` must satisfy: `total_reserves_usd ≥ minted_acbu_supply × RESERVE_MIN_RATIO`.

---

## 6. Deploy Backend Service

```bash
cd backend
npm ci --production
npx prisma generate
npm run build
npm run start:prod
```

Verify the service is healthy:

```bash
curl https://api.acbu.io/v1/health
```

Expected response: `{ "status": "ok" }`.

---

## 7. Post-Deployment: Org Limits Backfill

If this is an upgrade of an existing deployment (not a fresh launch), run the org-limits backfill to ensure existing transactions carry correct `rateSnapshot.organizationId`. Full procedure in [DEPLOYMENT_ORG_LIMITS_BACKFILL.md](./DEPLOYMENT_ORG_LIMITS_BACKFILL.md):

```bash
# Dry-run first
npx ts-node prisma/backfillOrgTransactionContext.ts

# Apply if candidate/derivable counts look correct
npx ts-node prisma/backfillOrgTransactionContext.ts --apply
```

---

## 8. Configure Fintech Webhooks

Register the ACBU backend webhook URL with each active payment provider so on-ramp and off-ramp events are delivered:

| Provider | Webhook URL |
|----------|-------------|
| Flutterwave | `https://api.acbu.io/v1/webhooks/flutterwave` |
| Paystack | `https://api.acbu.io/v1/webhooks/paystack` |
| MTN MoMo | `https://api.acbu.io/v1/webhooks/mtn` |

Set `WEBHOOK_SECRET` in `backend/.env` to the shared HMAC-SHA256 secret used to verify incoming payloads.

---

## 9. Smoke Tests

Run through the critical paths to confirm the deployment is functional:

1. **Oracle:** `GET /v1/rates` returns current basket rates.
2. **Reserve check:** `GET /v1/reserves` returns non-zero reserves with ratio ≥ `RESERVE_MIN_RATIO`.
3. **Mint (retail):** `POST /v1/retail/mint/deposit` with valid payload completes and returns a `Transaction` row with `status=completed` and a non-null `blockchain_tx_hash`.
4. **Burn (retail):** `POST /v1/retail/burn` with valid payload initiates a redemption.
5. **API key scopes:** Attempt a segment route (e.g. `POST /v1/p2p/transfer`) with an API key missing `p2p:write` — expect a `403`.
6. **Rate limiting:** Exceed `RATE_LIMIT_MAX_REQUESTS` in one window — expect `429`.

---

## 10. Monitoring and Alerts

- **Sentry:** Confirm errors are appearing in the Sentry project by triggering a test error.
- **Reserve ratio:** Set up an alert if `reserve_ratio` drops below `RESERVE_ALERT_THRESHOLD` (default 1.02).
- **Oracle staleness:** Alert if oracle rates have not been updated within `ORACLE_UPDATE_INTERVAL_HOURS × 2`.
- **RabbitMQ queue depth:** Alert on unusually deep queues (pending mint/burn jobs).

---

## 11. Rollback Procedure

If a deployment needs to be rolled back:

1. Revert the backend service to the previous build artifact.
2. Run `npx prisma migrate resolve --rolled-back <migration_name>` for any migration that should be reverted.
3. Smart contract deployments are **not reversible** — to roll back contract logic, deploy a new contract version and update `CONTRACT_*` env vars, then redeploy the backend.
4. If the org-limits backfill (`--apply`) ran against data that should be reverted, restore from a database snapshot taken before deployment.

---

## Related Documents

- [ENV_VARS.md](./ENV_VARS.md) — full environment variable reference
- [TESTNET_CUSTODIAL_BOOTSTRAP.md](./TESTNET_CUSTODIAL_BOOTSTRAP.md) — testnet-specific bootstrap steps
- [DEPLOYMENT_ORG_LIMITS_BACKFILL.md](./DEPLOYMENT_ORG_LIMITS_BACKFILL.md) — post-deploy org limits backfill
- [TECHNICAL/ARCHITECTURE.MD](./TECHNICAL/ARCHITECTURE.MD) — system architecture overview
- [TECHNICAL/DATABASE_SCHEMA.MD](./TECHNICAL/DATABASE_SCHEMA.MD) — database schema reference
