---
name: mithril
version: "0.1.0"
description: >
  Use this skill whenever the user asks to send USDC, make on-chain payments,
  check credit or spending power, manage crypto wallets, repay a credit line,
  view a balance sheet or ledger, interact with x402 services, or do anything
  involving the Mithril API. Also trigger when the user mentions "Mithril",
  "agent credit", "stablecoin payments", "USDC transfer", "crypto credit card",
  "on-chain payment", "pay with crypto", "agent wallet", or asks about on-chain
  spending limits, credit lines, or billing. Covers Base, Polygon, Arbitrum,
  Ethereum, and Solana networks. Use this skill even if you're unsure whether
  the task involves Mithril — if it sounds like agent payments, wallets, or
  credit, trigger this skill and check.
user-invocable: true
allowed-tools: Read, Grep, Glob, Bash
---

```
MITHRIL API QUICK REFERENCE v0.1.0

Base:  https://api.trymithril.com
Auth:  Authorization: Bearer <MITHRIL_API_KEY>
Docs:  This file is canonical.

Spending:
  GET  /v1/agent/spending-power              -> how much you can spend right now

Payments (x402):
  POST /v1/agent/pay                         -> pay x402 service (auto-selects wallet)
  POST /v1/wallets/:id/pay                   -> pay x402 service (specific wallet)

Transfers:
  POST /v1/wallets/:id/send                  -> send USDC to any on-chain address

Wallets:
  GET  /v1/wallets                           -> list your wallets
  GET  /v1/wallets/:id                       -> get a single wallet
  PUT  /v1/wallets/:id/primary               -> set primary wallet
  POST /v1/wallets/:id/addresses             -> add address on a new network

Credit (per wallet):
  POST /v1/wallets/:id/repay                 -> make a payment (full, minimum, or custom)
  POST /v1/wallets/:id/cash-advance          -> withdraw USDC to deposit address (3% fee)
  POST /v1/wallets/:id/reconcile             -> detect deposits and apply to balance
  GET  /v1/wallets/:id/transactions          -> transaction history
  GET  /v1/wallets/:id/billing               -> billing info for this wallet

Cards:
  GET  /v1/agent/cards                       -> list your virtual cards
  GET  /v1/agent/cards/:id/details           -> get card details (PAN, CVV, etc.)

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

- **Pay for services** — if a service supports x402 (HTTP 402 Payment Required), point Mithril at the URL. It handles payment negotiation, signing, and retry automatically. You get the service's response back. Each wallet has deposits (your on-chain USDC) and a credit line. Deposits are spent first, then credit kicks in automatically.
- **Send USDC** — transfer USDC from your wallet to any on-chain address. Only your deposit balance can be sent (not credit). Use cash advance first if you need to borrow and then send.
- **Check your spending power** — see deposits + available credit across all wallets.
- **Manage your credit** — make payments, view transactions. Pay your bill on time and you never pay interest.
- **Take a cash advance** — withdraw USDC from your credit line to your deposit address. 3% fee, interest starts immediately.
- **Track your finances** — balance sheet, ledger, credit profile, and transaction history.

## Setup

Your human operator signs up at the dashboard, creates your agent, sets up your wallets, and generates an API key for you. One human account can run multiple agents — each agent gets its own API key and its own wallets. Wallets are created and configured by your operator through the dashboard — you cannot create wallets yourself.

This skill is for **one specific agent**. If your operator runs multiple agents, each one has a separate `MITHRIL_API_KEY` and operates independently.

Store your API key:

```bash
export MITHRIL_API_KEY="mith_..."
```

Every request needs this header:

```
Authorization: Bearer <MITHRIL_API_KEY>
```

**Security:** Your API key is the only thing protecting your account. The key IS your card. **Never log it, print it, or include it in any output.** If compromised, your operator can revoke it from the dashboard.

---

## Trigger Phrases & Example Prompts

| User says | Endpoint |
|-----------|----------|
| "How much can I spend?" / "What's my spending power?" | `GET /v1/agent/spending-power` |
| "Pay my Mithril bill" / "Repay my credit line" | `POST /v1/wallets/:id/repay` |
| "Buy data from this API" / "Purchase from this x402 service" | `POST /v1/agent/pay` |
| "Send USDC to this address" / "Transfer funds" | `POST /v1/wallets/:id/send` |
| "Show me my balance sheet" | `GET /v1/balance-sheet` |
| "What's my credit score?" / "Show my credit profile" | `GET /v1/credit-profile` |
| "List my wallets" / "What wallets do I have?" | `GET /v1/wallets` |
| "Show my transaction history" | `GET /v1/wallets/:id/transactions` |
| "Show me my ledger" | `GET /v1/ledger` |
| "I need a cash advance" | `POST /v1/wallets/:id/cash-advance` |
| "Check for new deposits" | `POST /v1/wallets/:id/reconcile` |
| "Show my cards" | `GET /v1/agent/cards` |

---

## Endpoint Decision Guide

When the user's request could map to multiple endpoints, use this decision tree:

1. **Buying from a service that returns HTTP 402** → `POST /v1/agent/pay`
   - Mithril handles payment negotiation, signing, and retry automatically.

2. **Sending USDC to an address** → `POST /v1/wallets/:id/send`
   - Only deposit balance can be sent. Credit cannot be transferred directly.

3. **Need to use a specific wallet instead of primary** → use the `/v1/wallets/:id/…` variant of any endpoint.

4. **Checking if you can afford something** → `GET /v1/agent/spending-power`
   - Always do this before large purchases to avoid wasting a rate-limited call on insufficient funds.

5. **Paying the monthly bill** → `POST /v1/wallets/:id/repay`
   - Use `mode: "full"` to avoid interest. Use `mode: "minimum"` if the user can't pay in full.

6. **Withdrawing credit as USDC to the deposit address** → `POST /v1/wallets/:id/cash-advance`
   - **Avoid unless the user explicitly asks.** 3% fee + immediate interest.

---

## Typical Workflow: Paying an x402 Service

```bash
# Step 1: Check spending power (recommended before large payments)
curl -sS "https://api.trymithril.com/v1/agent/spending-power" \
  -H "Authorization: Bearer $MITHRIL_API_KEY"

# Step 2: Make the payment
curl -sS -X POST https://api.trymithril.com/v1/agent/pay \
  -H "Authorization: Bearer $MITHRIL_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://api.service.com/data",
    "method": "GET"
  }'

# Step 3: Use the response body directly — it's the raw service response
```

---

## Common Pitfalls

**Read these before making API calls.**

1. **Always check spending-power before a large purchase.** Spending endpoints are rate-limited (~10/min). Don't waste a call on an insufficient-funds failure — check `GET /v1/agent/spending-power` first.

2. **`response.body` from payments is the raw response.** After `POST /v1/agent/pay`, the `response.body` field contains the service's raw response (typically JSON). Use it directly.

3. **The `amount` field is a string, not a number.** Send `"25.00"` not `25.00`. The API rejects numeric values.

4. **Never log or print the API key.** The key IS the credential. No secondary authentication exists.

5. **Cash advances have a 3% fee and immediate interest.** Only use `POST /v1/wallets/:id/cash-advance` when the user explicitly asks.

6. **Rate limits are tight for spending endpoints (~10/min).** Batch your logic. Plan all purchases first, then execute. Read endpoints are more generous (~60/min).

7. **Deposits are spent before credit automatically.** You don't need to manage this — the response shows `deposit_used` and `credit_used`.

8. **Wallet IDs are required for per-wallet endpoints.** Get them from `GET /v1/wallets` first. Don't guess or hardcode.

---

## Error Handling

| HTTP Code | What Happened | What To Do |
|-----------|---------------|------------|
| **400** Bad Request | Missing or invalid fields. | Check request body. Common: missing field, wrong `network`, numeric `amount` instead of string. Fix and retry. |
| **401** Unauthorized | API key missing or invalid. | Ask the user to verify `MITHRIL_API_KEY`. Should start with `mith_`. Operator can generate a new one from the dashboard. |
| **403** Forbidden | Action disallowed. Wallet may be frozen. | Check `credit_status` on the wallet — if `"frozen"`, operator needs to resolve missed payments. Do not retry. |
| **404** Not Found | Resource doesn't exist. | Verify wallet ID with `GET /v1/wallets`. |
| **429** Too Many Requests | Rate limited. | Back off 5-10 seconds. **Do not retry spending endpoints more than once** — duplicate payments are worse than failures. |
| **500** Internal Error | Server failure. | Retry once after a few seconds. If it persists, tell user to contact team@trymithril.com. |

**Always surface `error.message` from the response to the user** — it's human-readable and explains what went wrong.

Error format:
```json
{
  "error": {
    "code": "bad_request",
    "message": "destination is required"
  }
}
```

---

## API Reference

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
  "total_spending_power": "5000.00",
  "wallets": [
    {
      "wallet_id": "uuid",
      "label": "Main Wallet",
      "is_primary": true,
      "deposit_balance": "500.00",
      "available_credit": "4500.00",
      "spending_power": "5000.00"
    }
  ]
}
```

---

### Pay

Auto-selects your primary wallet. Handles x402 payment negotiation automatically.

```
POST /v1/agent/pay
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `url` | string | yes | Target service URL |
| `method` | string | no | HTTP method (default `"GET"`) |
| `headers` | object | no | Custom headers as key-value pairs |
| `body` | string | no | Request body (base64 for binary, raw string otherwise) |

**Response (200):**
```json
{
  "status": "completed",
  "response": {
    "status_code": 200,
    "headers": {"content-type": "application/json"},
    "body": "{\"data\": [...]}"
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

- `response.body` is the raw response body from the service (typically JSON).
- `payment` is null if the service didn't require payment (no 402).

### Pay from Specific Wallet

```
POST /v1/wallets/:id/pay
```

Same request and response as above.

---

### Send

Transfer USDC from your wallet to any on-chain address. Only your deposit balance can be sent — credit cannot be transferred directly. If you need to borrow and send, use cash advance first.

```
POST /v1/wallets/:id/send
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `destination` | string | yes | On-chain address to send to |
| `amount` | string | yes | USDC amount (e.g. `"100.00"`) |
| `network` | string | yes | `"base"`, `"polygon"`, `"arbitrum"`, `"ethereum"`, or `"solana"` |

**Response (200):**
```json
{
  "tx_hash": "0x...",
  "amount": "100.000000",
  "network": "base",
  "deposit_used": "100.000000",
  "credit_used": "0.000000"
}
```

---

### Repay

Make a payment on your credit line.

```
POST /v1/wallets/:id/repay
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `mode` | string | yes | `"full"`, `"minimum"`, or `"custom"` |
| `amount` | string | conditional | Required when `mode` is `"custom"` |

**Response (200):** Returns the updated wallet object.

**Billing details:**
- **Billing day**: Statement generated showing what you owe.
- **Grace period**: 21 days. Pay in full = zero interest.
- **Minimum payment**: 2% of outstanding balance or $25, whichever is greater.
- **Auto-collection**: On the due date, the system auto-collects from your deposit balance.

### Cash Advance

Withdraw USDC from your credit line to your wallet's deposit address. **3% fee. Interest starts immediately (no grace period).**

```
POST /v1/wallets/:id/cash-advance
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `amount` | string | yes | USDC amount to withdraw |

**Response (200):** Returns the updated wallet object.

### Reconcile Deposits

Detect on-chain USDC deposits and apply them to your outstanding credit balance. The wallet's `auto_collect_mode` determines how much is applied (`full`, `minimum`, or `custom`).

```
POST /v1/wallets/:id/reconcile
```

No request body needed.

**Response (200):**

```json
{
  "amount_applied": "100.00",
  "remaining_balance": "250.00",
  "deposit_balance": "400.00",
  "tx_hash": "0x...",
  "network": "base",
  "auto_collect_mode": "full"
}
```

### Wallet Billing

Billing info for a wallet.

```
GET /v1/wallets/:id/billing
```

**Response (200):** Billing details for the wallet's current cycle.

### Wallet Transactions

All activity on a wallet's credit line.

```
GET /v1/wallets/:id/transactions?limit=50
```

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `limit` | number | no | Max results (default 50, max 100) |

**Response (200):**
```json
[
  {
    "id": "uuid",
    "agent_id": "uuid",
    "wallet_id": "uuid",
    "debit_account": "liability.credit.line",
    "credit_account": "asset.cash",
    "amount": "25.00",
    "currency": "USDC",
    "entry_type": "credit.spend",
    "reference_id": "uuid",
    "description": "Credit spend: 25.00 USDC",
    "network": "base",
    "tx_hash": "0x...",
    "merchant_domain": "api.service.com",
    "created_at": "2026-02-08T12:00:00Z"
  }
]
```

**Entry types:** `credit.spend`, `credit.repayment`, `credit.auto_collect`, `credit.interest`, `credit.cash_advance`, `credit.fee`, `transfer.send`

---

### Wallets

#### The Wallet Object

```json
{
  "id": "uuid",
  "label": "Main Wallet",
  "is_primary": true,
  "credit_limit": "5000.00",
  "apr": "0.20",
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
  "is_frozen": false,
  "auto_collect_mode": "full",
  "billing_day": 15,
  "grace_period_active": true,
  "missed_payments": 0,
  "created_at": "2026-01-15T10:00:00Z"
}
```

Key fields:
- `deposit_balance` — on-chain USDC in this wallet
- `available_credit` — how much more you can borrow
- `spending_power` — deposit_balance + available_credit
- `total_owed` — outstanding_balance + cash_advance_balance + accrued_interest
- `credit_status` — `"active"`, `"frozen"`, or `"closed"`
- `is_frozen` — whether the wallet is currently frozen
- `grace_period_active` — if true, new charges accrue no interest until the due date
- `missed_payments` — keep this at 0. Missing payments freezes your credit line.

#### List Wallets

```
GET /v1/wallets
```

**Response (200):** Array of wallet objects.

#### Get Wallet

```
GET /v1/wallets/:id
```

**Response (200):** Single wallet object.

#### Set Primary Wallet

The primary wallet is used by `POST /v1/agent/pay`.

```
PUT /v1/wallets/:id/primary
```

**Response (200):** The updated wallet object.

#### Add Wallet Address

Add an on-chain address on a new network to receive deposits.

```
POST /v1/wallets/:id/addresses
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `network` | string | yes | `"base"`, `"polygon"`, `"arbitrum"`, `"ethereum"`, or `"solana"` |

**Response (200):** The updated wallet object.

---

### Cards

Virtual cards linked to your agent for payments where x402 isn't supported.

#### List Cards

```
GET /v1/agent/cards
```

**Response (200):** Array of card objects.

#### Get Card Details

Retrieve full card details including PAN, CVV, and expiration.

```
GET /v1/agent/cards/:id/details
```

**Response (200):** Card object with sensitive details.

---

### Account & Reporting

#### Agent Profile

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

#### Agent Stats

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

#### Balance Sheet

Full view of assets, liabilities, and equity. Computed live.

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

#### Credit Profile

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
  "history_score": 1.05,
  "first_loan_at": "2026-01-20T10:00:00Z",
  "last_loan_at": "2026-02-05T14:30:00Z"
}
```

Tiers: `new` → `established` → `good` → `excellent` (or `risky` → `defaulter`)

#### Ledger

Full double-entry record of all financial activity.

```
GET /v1/ledger?entry_type=credit.spend&limit=20
```

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `entry_type` | string | no | Filter by type (see below) |
| `limit` | number | no | Max results (default 50) |

**Entry types:** `credit.spend`, `credit.repayment`, `credit.auto_collect`, `credit.interest`, `credit.cash_advance`, `credit.fee`, `credit.pool.funding`, `transfer.send`

**Response (200):**
```json
[
  {
    "id": "uuid",
    "agent_id": "uuid",
    "wallet_id": "uuid",
    "debit_account": "liability.credit.line",
    "credit_account": "asset.cash",
    "amount": "25.00",
    "currency": "USDC",
    "entry_type": "credit.spend",
    "reference_id": "uuid",
    "description": "Credit spend: 25.00 USDC",
    "network": "base",
    "tx_hash": "0x...",
    "merchant_domain": "api.service.com",
    "created_at": "2026-02-08T12:00:00Z"
  }
]
```

---

## Links

- **Sign up (human):** https://app.trymithril.com/signup
- **Dashboard:** https://app.trymithril.com
- **API base:** https://api.trymithril.com
- **Support:** team@trymithril.com
