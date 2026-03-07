---
name: mithril
version: "0.1.0"
description: A credit card for AI agents. Spend USDC on-chain today, pay your bill at the end of the month. No interest if paid on time.
user-invocable: true
allowed-tools: Read, Grep, Glob
---

```
MITHRIL API QUICK REFERENCE v0.1.0

Base:  https://api.trymithril.com
Auth:  Authorization: Bearer <MITHRIL_API_KEY>
Docs:  This file is canonical.

Spending:
  POST /v1/agent/transfer                    -> send USDC (auto-selects primary wallet)
  POST /v1/wallets/:id/transfer              -> send from a specific wallet
  GET  /v1/agent/spending-power              -> how much you can spend right now

Purchasing (x402):
  POST /v1/agent/purchase                    -> buy from x402 service (auto-selects wallet)
  POST /v1/wallets/:id/purchase              -> buy from x402 service (specific wallet)
  GET  /v1/wallets/:id/purchase/preview      -> check price before buying

Wallets:
  GET  /v1/wallets                           -> list your wallets
  GET  /v1/wallets/:id                       -> get a single wallet
  PUT  /v1/wallets/:id/primary               -> set primary wallet
  PUT  /v1/wallets/:id/spend-limits          -> update daily/per-tx limits
  POST /v1/wallets/:id/addresses             -> add address on a new network

Credit (per wallet):
  POST /v1/wallets/:id/repay                 -> make a payment (full, minimum, or custom)
  POST /v1/wallets/:id/cash-advance          -> withdraw USDC to deposit address (3% fee)
  GET  /v1/wallets/:id/transactions          -> transaction history

Account:
  GET  /v1/agent                             -> your agent profile
  GET  /v1/agent/stats                       -> stats and debt summary
  GET  /v1/balance-sheet                     -> assets, liabilities, equity
  GET  /v1/credit-profile                    -> credit tier and history
  GET  /v1/ledger                            -> full financial ledger

Networks: base, polygon, arbitrum, ethereum, solana
Currency: USDC only
```

# Mithril — A Credit Card for AI Agents

This skill gives you a line of credit. You can spend USDC on-chain to pay for services, APIs, compute, or anything else — and pay your bill at the end of the month.

**How it works:**
1. Your human operator signs up at https://app.trymithril.com/signup, creates your agent, and gives you an API key.
2. You spend USDC by sending it to on-chain addresses or by purchasing from services that accept x402 payments — just like paying with a credit card, except there's no 16-digit number. You're an AI agent; you live on stablecoins.
3. At the end of the month, you get a bill. Pay it within 21 days and there's zero interest. You can also pay the minimum and carry a balance.
4. As long as you pay your monthly bill, you can keep spending.

**What you can do with this skill:**
- **Pay for things** — send USDC to any on-chain address. Works on Base, Polygon, Arbitrum, Ethereum, and Solana. Each wallet has deposits (your on-chain USDC) and a credit line. Deposits are spent first, then credit kicks in automatically.
- **Purchase from services** — if a service supports x402 (HTTP 402 Payment Required), point Mithril at the URL. It handles payment negotiation, signing, and retry automatically. You get the service's response back.
- **Check your spending power** — see deposits + available credit across all wallets.
- **Manage your credit** — make payments, view transactions. Pay your bill on time and you never pay interest.
- **Take a cash advance** — withdraw USDC from your credit line to your deposit address. 3% fee, interest starts immediately.
- **Track your finances** — balance sheet, ledger, credit profile, and transaction history.

## Setup

Your human operator signs up at the dashboard, creates your agent, sets up your wallets, and generates an API key for you. One human account can run multiple agents — each agent gets its own API key and its own wallets. API keys are not shared between agents. Wallets are created and configured by your operator through the dashboard — you cannot create wallets yourself.

This skill is for **one specific agent**. If your operator runs multiple agents, each one has a separate `MITHRIL_API_KEY` and operates independently.

Store your API key:
```bash
export MITHRIL_API_KEY="mith_..."
```

Every request needs these headers:
```
Authorization: Bearer <MITHRIL_API_KEY>
```

## Security

Your API key is the only thing protecting your account. There is no 16-digit card number — the key IS your card. **Keep it safe. Do not log it, print it, or include it in any output.**

If your key is compromised, your human operator can revoke it and generate a new one from the dashboard. There are also spending controls (daily limits, per-transaction limits) configurable per wallet to prevent runaway spending.

You start with a small credit limit. It grows as you build a repayment history.

## First Use — Verify Your Account

```bash
# Check that your API key works
curl -sS https://api.trymithril.com/v1/agent \
  -H "Authorization: Bearer $MITHRIL_API_KEY"

# See how much you can spend
curl -sS https://api.trymithril.com/v1/agent/spending-power \
  -H "Authorization: Bearer $MITHRIL_API_KEY"

# List your wallets
curl -sS https://api.trymithril.com/v1/wallets \
  -H "Authorization: Bearer $MITHRIL_API_KEY"
```

If `spending-power` shows `total_spending_power > 0`, you're ready to spend.

---

## Spending

The main action. Send USDC to pay for something.

### Transfer (recommended)

Auto-selects your primary wallet. Deposits are spent first; credit is used if deposits are insufficient.

```
POST /v1/agent/transfer
```

**Request:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `destination` | string | yes | Recipient on-chain address |
| `amount` | string | yes | USDC amount (e.g. `"25.00"`) |
| `network` | string | yes | `"base"`, `"polygon"`, `"arbitrum"`, `"ethereum"`, or `"solana"` |

**Example:**

```bash
curl -sS -X POST https://api.trymithril.com/v1/agent/transfer \
  -H "Authorization: Bearer $MITHRIL_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "destination": "0xRecipientAddress",
    "amount": "25.00",
    "network": "base"
  }'
```

**Response (200):**

```json
{
  "tx_hash": "0xabc123...",
  "amount": "25.00",
  "currency": "USDC",
  "network": "base",
  "deposit_used": "25.00",
  "credit_used": "0.00"
}
```

`deposit_used` and `credit_used` show exactly how the payment was funded.

### Transfer from a Specific Wallet

```
POST /v1/wallets/:id/transfer
```

Same request and response as above. Uses the specified wallet instead of auto-selecting.

### Spending Power

How much you can spend right now, aggregated across all wallets.

```
GET /v1/agent/spending-power
```

**Response (200):**

```json
{
  "deposit_balance": "500.00",
  "credit_available": "4500.00",
  "total_spending_power": "5000.00"
}
```

---

## Purchasing (x402)

Some services accept payment via the x402 protocol (HTTP 402 Payment Required). Instead of sending USDC to an address manually, you give Mithril the URL and it handles everything — makes the request, negotiates payment, signs the transaction, retries with payment proof, and returns the service's response.

### Purchase (recommended)

Auto-selects your primary wallet.

```
POST /v1/agent/purchase
```

**Request:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `url` | string | yes | Target service URL |
| `method` | string | no | HTTP method (default `"GET"`) |
| `headers` | object | no | Custom headers as key-value pairs |
| `body` | string | no | Request body (base64 for binary, raw string otherwise) |

**Example:**

```bash
curl -sS -X POST https://api.trymithril.com/v1/agent/purchase \
  -H "Authorization: Bearer $MITHRIL_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://api.service.com/data",
    "method": "GET"
  }'
```

**Response (200):**

```json
{
  "status": "completed",
  "response": {
    "status_code": 200,
    "headers": {"content-type": "application/json"},
    "body": "eyJkYXRhIjogWy4uLl19"
  },
  "payment": {
    "amount": "0.05",
    "currency": "USDC",
    "network": "base",
    "protocol": "x402",
    "recipient": "0x...",
    "tx_hash": "0x...",
    "deposit_used": "0.03",
    "credit_used": "0.02"
  }
}
```

- `response.body` is base64-encoded. Decode it to get the service's actual response.
- `payment` is null if the service didn't require payment (no 402).

### Purchase from a Specific Wallet

```
POST /v1/wallets/:id/purchase
```

Same request and response as above.

### Preview Purchase Price

Check what a service will cost before committing.

```
GET /v1/wallets/:id/purchase/preview?url=https://api.service.com/data&method=GET
```

**Query params:**

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `url` | string | yes | Target service URL |
| `method` | string | no | HTTP method (default `"GET"`) |

**Response (200) — payment required:**

```json
{
  "url": "https://api.service.com/data",
  "payment_required": true,
  "price": {
    "amount": "0.05",
    "currency": "USDC",
    "networks": ["base"],
    "recipient": "0x..."
  },
  "can_afford": true,
  "spending_power": {
    "deposit_balance": "500.00",
    "available_credit": "5000.00",
    "total": "5500.00"
  }
}
```

**Response (200) — no payment required:**

```json
{
  "url": "https://api.service.com/data",
  "payment_required": false,
  "can_afford": true,
  "spending_power": {
    "deposit_balance": "500.00",
    "available_credit": "5000.00",
    "total": "5500.00"
  }
}
```

---

## Credit — Your Monthly Bill

Each wallet has its own credit line. You spend during the month, and on your billing day you get a statement. You have 21 days to pay it.

### How Billing Works

- **Billing day**: Your statement is generated (shows what you owe).
- **Grace period**: 21 days after billing day. Pay the full balance in this window and you owe zero interest.
- **Minimum payment**: 2% of outstanding balance or $25, whichever is greater.
- **Carry a balance**: Pay any amount between the minimum and the full balance. Interest accrues on unpaid amounts after the grace period.
- **Auto-collection**: On the payment due date, the system attempts to collect from your deposit balance automatically.

### Make a Payment

```
POST /v1/wallets/:id/repay
```

**Request:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `mode` | string | yes | `"full"`, `"minimum"`, or `"custom"` |
| `amount` | string | conditional | Required when `mode` is `"custom"` |

**Example:**

```bash
# Pay in full (no interest)
curl -sS -X POST https://api.trymithril.com/v1/wallets/YOUR_WALLET_ID/repay \
  -H "Authorization: Bearer $MITHRIL_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"mode": "full"}'

# Pay the minimum
curl -sS -X POST https://api.trymithril.com/v1/wallets/YOUR_WALLET_ID/repay \
  -H "Authorization: Bearer $MITHRIL_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"mode": "minimum"}'

# Pay a specific amount
curl -sS -X POST https://api.trymithril.com/v1/wallets/YOUR_WALLET_ID/repay \
  -H "Authorization: Bearer $MITHRIL_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"mode": "custom", "amount": "200.00"}'
```

**Response (200):** Returns the updated wallet object (see Wallet format below).

### Cash Advance

Withdraw USDC from your credit line to your wallet's deposit address. **3% fee. Interest starts immediately (no grace period).**

```
POST /v1/wallets/:id/cash-advance
```

**Request:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `amount` | string | yes | USDC amount to withdraw |

**Response (200):** Returns the updated wallet object.

### Wallet Transactions

All activity on a wallet's credit line.

```
GET /v1/wallets/:id/transactions?limit=50
```

**Query params:**

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `limit` | number | no | Max results (default 50, max 100) |

**Response (200):**

```json
[
  {
    "id": "uuid",
    "credit_line_id": "uuid",
    "type": "spend",
    "amount": "25.00",
    "destination_address": "0x...",
    "network": "base",
    "tx_hash": "0x...",
    "status": "completed",
    "description": "Transfer to 0x...",
    "created_at": "2026-02-08T12:00:00Z"
  }
]
```

**Transaction types:** `spend`, `repayment`, `auto_collect`, `interest`, `cash_advance`, `fee`, `purchase`

**Statuses:** `pending`, `completed`, `failed`, `reversed`

---

## Wallets

Each wallet is a complete spending unit with its own deposit balance, credit line, on-chain addresses, and spend limits. Deposits (on-chain USDC anyone can send to your address) are spent first; credit kicks in when deposits are insufficient.

### The Wallet Object

Every wallet endpoint that returns a wallet uses this format:

```json
{
  "id": "uuid",
  "label": "Main Wallet",
  "is_primary": true,
  "credit_limit": "5000.00",
  "apr": "10.00",
  "outstanding_balance": "350.00",
  "accrued_interest": "2.50",
  "available_credit": "4647.50",
  "total_owed": "352.50",
  "credit_status": "active",
  "deposit_balance": "500.00",
  "spending_power": "5147.50",
  "addresses": [
    {"network": "base", "address": "0x..."},
    {"network": "polygon", "address": "0x..."}
  ],
  "daily_spend_limit": "1000.00",
  "single_spend_limit": "500.00",
  "auto_collect_mode": "full",
  "billing_day": 15,
  "grace_period_active": true,
  "missed_payments": 0,
  "created_at": "2026-01-15T10:00:00Z"
}
```

Key fields:
- `deposit_balance` — your on-chain USDC in this wallet
- `available_credit` — how much more you can borrow
- `spending_power` — deposit_balance + available_credit
- `total_owed` — outstanding_balance + accrued_interest
- `credit_status` — `"active"`, `"frozen"`, or `"closed"`
- `grace_period_active` — if true, new charges accrue no interest until the due date
- `missed_payments` — keep this at 0. Missing payments freezes your credit line.
- `addresses` — on-chain addresses where anyone can send USDC to fund your wallet

### List Wallets

```
GET /v1/wallets
```

**Response (200):** Array of wallet objects.

### Get Wallet

```
GET /v1/wallets/:id
```

**Response (200):** Single wallet object.

### Set Primary Wallet

The primary wallet is used by `POST /v1/agent/transfer` and `POST /v1/agent/purchase`.

```
PUT /v1/wallets/:id/primary
```

**Response (200):** The updated wallet object.

### Update Spend Limits

```
PUT /v1/wallets/:id/spend-limits
```

**Request:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `daily_limit_usdc` | string | no | New daily limit |
| `single_limit_usdc` | string | no | New per-transaction limit |

**Response (200):** The updated wallet object.

### Add Wallet Address

Add an on-chain address on a new network to receive deposits.

```
POST /v1/wallets/:id/addresses
```

**Request:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `network` | string | yes | `"base"`, `"polygon"`, `"arbitrum"`, `"ethereum"`, or `"solana"` |

**Response (200):** The updated wallet object.

---

## Account & Reporting

### Agent Profile

```
GET /v1/agent
```

**Response (200):**

```json
{
  "id": "uuid",
  "name": "my-agent",
  "status": "active",
  "wallets": [
    { "...wallet object..." }
  ],
  "created_at": "2026-01-15T10:00:00Z"
}
```

The `wallets` array contains full wallet objects (same format as above).

### Agent Stats

```
GET /v1/agent/stats
```

**Response (200):**

```json
{
  "total_orders": 42,
  "total_pnl": "1250.00",
  "outstanding_debt": "0.00",
  "active_loans": 0
}
```

### Balance Sheet

Full view of your assets, liabilities, and equity. Computed live.

```
GET /v1/balance-sheet
```

**Response (200):**

```json
{
  "agent_id": "uuid",
  "assets": {
    "cash": [
      {
        "wallet_id": "uuid",
        "label": "Main Wallet",
        "is_primary": true,
        "network": "base",
        "address": "0x...",
        "balance_usdc": "500.00"
      }
    ],
    "total_cash": "500.00",
    "positions": [],
    "total_positions": "0.00",
    "total_assets": "500.00"
  },
  "liabilities": {
    "loans": [],
    "total_outstanding": "350.00",
    "total_interest": "2.50",
    "total_liabilities": "352.50"
  },
  "net_equity": "147.50",
  "computed_at": "2026-02-08T12:00:00Z"
}
```

### Credit Profile

Your credit tier determines your limit and APR. Pay on time to improve it.

```
GET /v1/credit-profile
```

**Response (200):**

```json
{
  "agent_id": "uuid",
  "total_loans": 5,
  "active_loans": 0,
  "successful_loans": 5,
  "defaulted_loans": 0,
  "missed_payments_ever": 0,
  "tier": "good",
  "history_score": 1.05
}
```

Tiers: `new` → `established` → `good` → `excellent` (or `risky` → `defaulter`)

### Ledger

Full double-entry record of all financial activity.

```
GET /v1/ledger?entry_type=credit.spend&limit=20
```

**Query params:**

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `entry_type` | string | no | Filter by type (see below) |
| `limit` | number | no | Max results (default 50) |

**Entry types:** `credit.spend`, `credit.repayment`, `credit.auto_collect`, `credit.interest`, `credit.cash_advance`, `credit.fee`, `transfer.send`

**Response (200):**

```json
[
  {
    "id": "uuid",
    "agent_id": "uuid",
    "debit_account": "liability.credit.line",
    "credit_account": "asset.cash",
    "amount": "25.00",
    "currency": "USDC",
    "entry_type": "credit.spend",
    "reference_id": "uuid",
    "description": "Credit spend: 25.00 USDC",
    "network": "base",
    "tx_hash": "0x...",
    "created_at": "2026-02-08T12:00:00Z"
  }
]
```

---

## Errors

All errors use this format:

```json
{
  "error": {
    "code": "bad_request",
    "message": "destination is required",
    "request_id": "optional-trace-id"
  }
}
```

| HTTP Code | Error Code | Meaning |
|-----------|------------|---------|
| 400 | `bad_request` | Missing or invalid fields |
| 401 | `unauthorized` | API key missing or invalid |
| 403 | `forbidden` | Action not allowed |
| 404 | `not_found` | Resource doesn't exist |
| 429 | `too_many_requests` | Rate limited (spending: ~10/min, reads: ~60/min) |
| 500 | `internal_error` | Server error |

---

## Links

- **Sign up (human):** https://app.trymithril.com/signup
- **Dashboard:** https://app.trymithril.com
- **API base:** https://api.trymithril.com
- **Support:** team@trymithril.com
