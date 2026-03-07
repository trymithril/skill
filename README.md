# Mithril — Credit Card for AI Agents

Give your AI agent a line of credit. Spend USDC on-chain today, pay the bill at the end of the month.

## Install

**Claude Code**
```bash
claude skills add https://trymithril.com/SKILL.md
```

**OpenClaw**
```bash
npx clawdhub@latest install mithril
```

**OpenAI Codex CLI**
```bash
codex skills add https://trymithril.com/SKILL.md
```

**Manual**
```bash
mkdir -p ~/.claude/skills/mithril
curl -o ~/.claude/skills/mithril/SKILL.md https://trymithril.com/SKILL.md
```

## What the skill exposes

Once loaded, your agent has access to the full Mithril API.

### Spending

`POST /v1/agent/transfer` — send USDC to any on-chain address. Auto-selects your primary wallet; deposits are used first, credit kicks in automatically when they run out.

`POST /v1/wallets/:id/transfer` — same, but from a specific wallet.

`GET /v1/agent/spending-power` — total deposits + available credit across all wallets.

### x402 Purchasing

`POST /v1/agent/purchase` — buy from any service that uses the [x402](https://x402.org) payment protocol. Pass a URL; Mithril handles negotiation, signing, payment, and retry. Returns the service's decoded response.

`GET /v1/wallets/:id/purchase/preview` — check what a service will cost before committing.

### Wallets

`GET /v1/wallets` — list all wallets with balances, credit lines, and on-chain addresses.

`PUT /v1/wallets/:id/spend-limits` — set daily and per-transaction limits.

`POST /v1/wallets/:id/addresses` — add an address on a new network to receive USDC deposits.

### Credit

`POST /v1/wallets/:id/repay` — make a payment: `full`, `minimum`, or `custom` amount.

`POST /v1/wallets/:id/cash-advance` — withdraw USDC from the credit line to your deposit address (3% fee, interest starts immediately).

`GET /v1/wallets/:id/transactions` — full transaction history for a wallet.

### Account & Reporting

`GET /v1/agent` — agent profile and all wallets.

`GET /v1/agent/stats` — total orders, PnL, outstanding debt.

`GET /v1/balance-sheet` — assets, liabilities, and net equity, computed live.

`GET /v1/credit-profile` — credit tier (`new` → `established` → `good` → `excellent`) and repayment history.

`GET /v1/ledger` — full double-entry financial ledger, filterable by entry type.

### Networks & currency

Base · Polygon · Arbitrum · Ethereum · Solana — USDC only.

## Setup

Sign up at [app.trymithril.com](https://app.trymithril.com/signup), create an agent, and set the API key:

```bash
export MITHRIL_API_KEY="mith_..."
```

## SDKs

Prefer to integrate in code?

- **TypeScript / Node.js** — [github.com/trymithril/typescript-sdk](https://github.com/trymithril/typescript-sdk)
- **Python** — [github.com/trymithril/python-sdk](https://github.com/trymithril/python-sdk)

---

[trymithril.com](https://trymithril.com) · [team@trymithril.com](mailto:team@trymithril.com) · *Alpha. Credit involves risk.*
