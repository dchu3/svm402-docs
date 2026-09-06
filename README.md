# SVM402 — Agent-Native Solana Token Analysis API

**Pay per call in USDC. No API keys. No signup. AI agents discover, pay for, and consume token analysis data autonomously via the x402 protocol.**

**What it tells you — and what it doesn't:** svm402 reads on-chain data to determine which tokens to **avoid** — honeypots, live mint or freeze authority, unlocked liquidity, wash-trading patterns, manufactured volume. It helps eliminate the cheats so the market is the only thing left to beat. It does not predict which way a token's price will move — that's impossible, for any tool.

**`/discover` inverts the filter** — the genuinely-traded tokens rise to the top by organic score. It surfaces demand authenticity, **not endorsements**: results are not recommendations to buy, and every discovered token still requires independent analysis (`/safety`, `/analyze`) before any decision.

![Solana Mainnet](https://img.shields.io/badge/Solana-Mainnet-9945FF?logo=solana&logoColor=white)
![x402 v2](https://img.shields.io/badge/x402-v2-blue)
![Own Facilitator Backup](https://img.shields.io/badge/Facilitator-Own_%2B_CDP_%2F_PayAI_Failover-14F195)
![MCP Compatible](https://img.shields.io/badge/MCP-Compatible-green)

- **Base URL:** https://svm402.com
- **Network:** Solana Mainnet
- **Payment:** x402 protocol — USDC SPL transfers
- **Seller wallet:** `4ofUMGcRHsfr6LMY6AuaRSQpqyKsvibSbXmieQL4i8Dc`
- **USDC mint:** `EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v`
- **Facilitators:** Coinbase CDP primary → self-hosted [SVM402 Facilitator](https://facilitator.svm402.com) (backup, sovereignty insurance) → PayAI — automatic failover chain

---

## Why svm402?

Solana token data is noisy. Most aggregators report inflated volume and holder counts driven by **wash trading**, and LLMs/agents that rely on them make bad decisions. svm402 is different:

- **Wash trading detection** — organic-token discovery and analysis filtered by on-chain organic scores and pool-vault behavior, not reported volume.
- **On-chain data accuracy** — holder counts, mint/freeze authority, taxes, and honeypot checks come straight from RPC (Helius), not cached third-party snapshots.
- **Built for agents** — x402-native payment, machine-readable discovery files, OpenAPI spec, and an MCP server. An agent can discover, pay for, and consume the API with zero human setup.
- **Pay per call** — priced in USDC micropayments. No accounts, no keys, no rate-limit tiers.

---

## Endpoints

| Method | Path | Cost | Description |
|--------|------|------|-------------|
| POST   | `/analyze` | $0.05 USDC | Full token analysis: price, liquidity, safety, holders, wash trading detection |
| GET    | `/analyze/{address}` | $0.05 USDC | Full analysis (GET variant) |
| GET    | `/safety/{address}` | $0.02 USDC | Honeypot / safety risk check with 0–10 risk score |
| GET    | `/wash-trading/{address}` | $0.02 USDC | Wash trading detection — bot volume manipulation indicators |
| GET    | `/price/{address}` | $0.01 USDC | Token price, liquidity, market cap, 24h volume, price changes |
| POST   | `/discover` | $0.02 USDC | Organic Solana token discovery — filtered to exclude wash trading |
| GET    | `/wallet/analyze/{address}` | $0.02 USDC | Wallet holdings: total value, SOL balance, top 20 tokens, risk summary |
| GET    | `/health` | Free | Service health check |

---

## Risk Score

Every safety response includes a deterministic **risk score from 0 to 10**:

| Range | Label |
|-------|-------|
| 0–2   | `low` |
| 2–4   | `moderate` |
| 4–6   | `high` |
| 6–8   | `very_high` |
| 8–10  | `extreme` |

**Factors evaluated:**
- Mint authority (revoked or live)
- Freeze authority (revoked or live)
- Liquidity locked
- Holder concentration
- Honeypot indicators
- Buy/sell tax

0 = safest, 10 = riskiest.

---

## Payment Flow (x402)

svm402 uses the [x402 protocol](https://www.x402.org) for per-request USDC micropayments on Solana:

1. **Make a request without payment** → receive `HTTP 402 Payment Required` with payment details (amount, recipient, mint).
2. **Send USDC** to the seller wallet (`4ofUMGcRHsfr6LMY6AuaRSQpqyKsvibSbXmieQL4i8Dc`) via a standard Solana SPL token transfer.
3. **Retry the request** with the header:
   ```
   X-Payment: <transaction_signature>
   ```
4. The server verifies the transaction on-chain (via the active facilitator in the chain — Coinbase CDP primary, our self-hosted [SVM402 Facilitator](https://facilitator.svm402.com) as backup, PayAI last) and returns the data.

No accounts. No API keys. The payment *is* the auth.

---

## Curl Examples

All paid endpoints return `402` without payment; examples show the happy path.

```bash
# Full token analysis (POST)
curl -X POST https://svm402.com/analyze \
  -H "Content-Type: application/json" \
  -H "X-Payment: <tx_signature>" \
  -d '{"address": "DezXAZ8z7PnrnRJjz3wXBoRgixCa6xjnB7YaB1pPB263"}'

# Full token analysis (GET)
curl https://svm402.com/analyze/DezXAZ8z7PnrnRJjz3wXBoRgixCa6xjnB7YaB1pPB263 \
  -H "X-Payment: <tx_signature>"

# Safety / honeypot check
curl https://svm402.com/safety/DezXAZ8z7PnrnRJjz3wXBoRgixCa6xjnB7YaB1pPB263 \
  -H "X-Payment: <tx_signature>"

# Wash trading detection
curl https://svm402.com/wash-trading/DezXAZ8z7PnrnRJjz3wXBoRgixCa6xjnB7YaB1pPB263 \
  -H "X-Payment: <tx_signature>"

# Price, liquidity, volume
curl https://svm402.com/price/DezXAZ8z7PnrnRJjz3wXBoRgixCa6xjnB7YaB1pPB263 \
  -H "X-Payment: <tx_signature>"

# Organic token discovery
curl -X POST https://svm402.com/discover \
  -H "Content-Type: application/json" \
  -H "X-Payment: <tx_signature>" \
  -d '{"limit": 10}'

# Wallet analysis
curl https://svm402.com/wallet/analyze/37fMqbe7vNoDuE15B1qas8TZyhqvYwgLVSViRa4DEmwa \
  -H "X-Payment: <tx_signature>"

# Health check (free)
curl https://svm402.com/health
```

---

## Discovery Files

svm402 is fully self-describing — point any crawler or agent at the root:

| File | Purpose |
|------|---------|
| `/.well-known/x402` | x402 payment discovery metadata |
| `/.well-known/ai-catalog.json` | ARD agent catalog |
| `/.well-known/endpoints.json` | Machine-readable endpoint list with prices |
| `/llms.txt` | LLM-friendly reference with curl examples |
| `/openapi.json` | OpenAPI 3.x spec |
| `/docs` | Interactive Swagger UI |
| `/robots.txt` | Crawler guidance |

---

## MCP Server

Use svm402 from any MCP-compatible client (Claude, Cursor, custom agents):

👉 **https://github.com/dchu3/svm402-mcp**

Six tools exposed:

- `analyze_token` — full token analysis
- `wallet_analyze` — wallet holdings breakdown
- `discover_tokens` — organic token discovery
- `check_safety` — honeypot/risk check
- `check_wash_trading` — wash trading detection
- `get_price` — price & liquidity

Payment is handled automatically under the hood via x402.

---

## Data Sources

- **Jupiter v3/v2 API** — price, market cap, liquidity, organic score
- **Helius RPC** — pool vaults, holder distribution, on-chain safety signals
- **pump.fun bonding curve** — price fallback for pre-migration tokens

All data is fetched at request time — nothing stale, nothing cached from third-party aggregators.

---

## Built for the Machine Economy

svm402 isn't a dashboard with an API bolted on — it's designed from the ground up for autonomous consumers. An AI agent can:

1. Find the service via `/.well-known/ai-catalog.json` or `llms.txt`
2. Read prices from `/.well-known/endpoints.json`
3. Pay in USDC via x402 — no human in the loop
4. Consume structured JSON and act on it

This is what the agent economy's data layer looks like.

---

## Disclaimer

Crypto assets are volatile and risky. svm402 provides data and analysis for informational purposes only — it is **not financial advice**. Always do your own research before making any transaction.

---

## Links

- **Service:** https://svm402.com
- **x402 protocol:** https://www.x402.org
- **MCP:** https://modelcontextprotocol.io
- **MCP server repo:** https://github.com/dchu3/svm402-mcp
