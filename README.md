# Fluxa

**Cross-border payment infrastructure for emerging markets.**

Fluxa is a programmable payments API built on the [Stellar](https://stellar.org) network. It gives fintech products and developers the primitives to move value across borders — wallet management, internal transfers, FX conversion via Stellar path payments, and settlement — behind a clean REST API.

> **Status**: Active development — testnet only.

---

## What it does

- **Wallets** — create Stellar accounts with AES-256-GCM encrypted secrets; never expose raw keys
- **Transfers** — async payment submission with queue-backed retry and status polling
- **FX / Conversion** — quote and execute cross-asset swaps via Stellar DEX path payments
- **Settlement** — background worker submits transactions to Stellar, handles retries, confirms on-chain
- **Ledger indexer** — streams Horizon events to keep local state in sync
- **Multi-tenant** — API key auth; individual developers and business organizations each get scoped access
- **Webhooks** — signed delivery of payment events to developer endpoints
- **Sandbox mode** — `sk_test_` keys route to Stellar testnet for safe integration testing

---

## Architecture

```
Client Applications
        │  Authorization: Bearer sk_live_... or sk_test_...
        ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Fluxa REST API                             │
│  Chi router │ JWT + API key auth │ Rate limiting │ Tenant scope │
└─────────────────────────────────────────────────────────────────┘
        │
        ├── Wallet Service       ──► postgres: wallets, balances
        ├── Transfer Service     ──► postgres: transactions
        ├── FX Service           ──► Stellar DEX + rate cache (Redis)
        ├── Fee Service          ──► postgres: fees, fee_collections
        └── Webhook Dispatcher   ──► postgres: webhook_endpoints, deliveries
                │
                ▼  (Asynq job queue)
        ┌────────────────────────────────────────┐
        │           Background Worker             │
        │  Settlement Engine │ Ledger Indexer     │
        │  Reconciliation    │ Scheduler          │
        └────────────────────────────────────────┘
                │
                ▼
        Stellar Network (Horizon API + Soroban RPC)
        testnet: horizon-testnet.stellar.org
        mainnet: horizon.stellar.org
```

**Two processes**

| Binary | Role |
|---|---|
| `cmd/api` | HTTP server — handles all REST requests, enqueues async work |
| `cmd/worker` | Asynq worker — settles transfers, runs ledger indexer, processes webhooks |

Transfers are **asynchronous**. `POST /v1/transfers` returns `202 Accepted` with a `pending` transaction immediately. Poll `GET /v1/transfers/:id` or receive a `transfer.settled` webhook for the final status.

---

## Project Structure

```
fluxa/
├── cmd/
│   ├── api/main.go           # HTTP server entry point
│   └── worker/main.go        # Background worker entry point
├── internal/
│   ├── config/               # Viper env config
│   ├── domain/               # Core types: Wallet, Transaction, Conversion, errors
│   ├── crypto/               # AES-256-GCM encrypt/decrypt (stdlib only)
│   ├── assets/               # Asset registry: USDC/EURC issuers per network
│   ├── stellar/              # Horizon client, keypair generation, signer interface
│   ├── postgres/             # pgx/v5 repository implementations
│   ├── queue/                # Asynq client + task type definitions
│   ├── wallet/               # Wallet service + HTTP handler
│   ├── transfer/             # Transfer service + HTTP handler
│   ├── fx/                   # FX service + rate providers + HTTP handler
│   ├── fees/                 # Fee calculation and collection
│   ├── settlement/           # Settlement engine + Asynq task handler
│   ├── indexer/              # Ledger indexer + Asynq periodic task
│   ├── webhook/              # Webhook dispatcher + delivery worker
│   ├── reconcile/            # DB vs on-chain reconciliation
│   ├── apikey/               # API key generation, hashing, verification
│   ├── auth/                 # User registration, login, JWT
│   ├── org/                  # Organization members, roles
│   ├── fiat/                 # Fiat rail abstraction + provider adapters
│   ├── server/               # Chi router setup, middleware
│   └── api/                  # Shared request validation + response helpers
└── db/
    └── migrations/           # golang-migrate SQL files (numbered up/down pairs)
```

---

## Getting Started

### Prerequisites

- Go 1.22+
- PostgreSQL 15+
- Redis 7+

### 1. Clone and install

```bash
git clone https://github.com/Savitura/Fluxa
cd Fluxa
go mod tidy
```

### 2. Configure environment

```bash
cp .env.example .env
```

| Variable | Description |
|---|---|
| `PORT` | HTTP listen port (default: `3000`) |
| `DATABASE_URL` | PostgreSQL connection string |
| `REDIS_URL` | Redis connection string |
| `STELLAR_NETWORK` | `testnet` or `mainnet` |
| `STELLAR_HORIZON_URL` | Horizon endpoint |
| `STELLAR_USDC_ISSUER` | USDC issuer public key |
| `MASTER_ENCRYPTION_KEY` | 64 hex chars (32 bytes) — encrypts stored wallet secrets |
| `PLATFORM_FEE_WALLET_PUBLIC_KEY` | Stellar address where platform fees are collected |
| `TREASURY_SECRET_KEY` | Stellar key that funds new accounts (testnet: leave empty, use Friendbot) |

Generate a master key:
```bash
openssl rand -hex 32
```

### 3. Run migrations

```bash
make migrate
```

### 4. Start the API and worker

```bash
# Terminal 1
make run-api

# Terminal 2
make run-worker
```

---

## API Reference

All endpoints are prefixed `/v1`. Auth: `Authorization: Bearer <api_key>`. Errors:

```json
{ "error": { "code": "WALLET_NOT_FOUND", "message": "wallet not found" } }
```

### Authentication

```http
POST /v1/auth/register     Create account (individual or organization)
POST /v1/auth/login        Login — returns JWT
POST /v1/keys              Create API key  →  sk_live_... or sk_test_...
GET  /v1/keys              List keys (prefix only, never raw)
DELETE /v1/keys/:id        Revoke key
```

### Wallets

```http
POST /v1/wallets           Create wallet — returns public key only
GET  /v1/wallets/:id       Get wallet
GET  /v1/wallets/:id/balances   Live balances from Horizon (all assets)
POST /v1/wallets/:id/trustlines  Add Stellar trustline for a new asset
```

### Transfers

```http
POST /v1/transfers         Initiate transfer (202 Accepted — async)
GET  /v1/transfers/:id     Poll status
GET  /v1/transfers         List (filter by wallet, status, date)
POST /v1/transfers/batch   Up to 100 transfers in one call
```

**Status flow:** `pending` → `confirmed` | `failed`

### FX

```http
POST /v1/fx/quote          Get a 30-second exchange rate quote
POST /v1/fx/convert        Execute a currency swap
GET  /v1/fx/rates          Live rates for a currency pair
```

### Webhooks

```http
POST   /v1/webhooks        Register endpoint + event subscriptions
GET    /v1/webhooks        List endpoints
DELETE /v1/webhooks/:id    Remove endpoint
GET    /v1/webhooks/:id/deliveries  Delivery log
```

**Event types:** `transfer.initiated` · `transfer.settled` · `transfer.failed` · `wallet.funded` · `conversion.completed`

### Usage

```http
GET /v1/usage              Current period: request count, transfer volume, rate limit
GET /v1/fees               Your fee schedule
GET /health                Health check
```

---

## Security

- **Key storage**: Stellar secrets are encrypted with AES-256-GCM before storage. The 32-byte master key lives only in env — never in the database or logs.
- **No key exposure**: Secret keys are never returned by any API endpoint.
- **Signer abstraction**: `stellar.Signer` in `internal/stellar/signer.go` isolates all signing. Swap `EnvSigner` for HSM or AWS KMS without touching the settlement engine.
- **Decimal arithmetic**: All monetary values use `shopspring/decimal` — no floating-point.
- **API key hashing**: Raw keys are SHA-256 hashed before storage; the plaintext is shown exactly once on creation.

---

## Development

```bash
make test          # go test ./... -race
make test-cover    # with HTML coverage report
make lint          # golangci-lint
make build         # outputs bin/api + bin/worker
make tidy          # go mod tidy
```

Fund a testnet wallet:
```bash
curl "https://friendbot.stellar.org?addr=<PUBLIC_KEY>"
```

---

## Part of Savitura

- **[CrowdPay](https://github.com/Savitura/crowdpay)** — crowdfunding platform built on top of Fluxa payment rails
- **[SaviTools](https://github.com/Savitura/Savitools)** — developer tools: API playground, transaction inspector, wallet sandbox

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

MIT
