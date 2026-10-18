# SVM402 — the Solana x402 Payment Lab for AI agents

**USDC per call. No API keys. No signup.** Agents discover the lab, settle with a USDC micro-payment via the x402 protocol, and consume — autonomously.

![Solana Devnet + Mainnet](https://img.shields.io/badge/Solana-Devnet_%2B_Mainnet-9945FF?logo=solana&logoColor=white)
![x402 v2](https://img.shields.io/badge/x402-v2-blue)
![Facilitator Failover](https://img.shields.io/badge/Facilitator-Failover-14F195)
![MCP Compatible](https://img.shields.io/badge/MCP-Compatible-green)

SVM402 is the **Solana x402 conformance and resilience lab** for AI agents: 14 deterministic x402 failure scenarios on the real money path, free payment diagnosis, paid on-chain settlement receipts. Built for **agent builders**, **x402 client SDK authors**, **self-facilitating sellers**, and **agents doing preflight** — everyone whose money is on the line the moment a request carries a payment. Rehearse the challenge, verify, settle and receipt machinery on devnet, then on mainnet — the lab's enforce/settle path IS the production gateway's, so the rehearsal is the production behavior, not a simulation.

- **Base URL:** https://svm402.com · devnet lab: https://lab-devnet.svm402.com
- **Network per challenge:** the server's CAIP-2 id — Solana mainnet `solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp` or devnet `solana:EtWTRABZaYq6iMfeYKouRu166VU2xqa1`
- **Asset:** USDC (6 decimals; integer base-unit strings over the wire)
- **Payment:** x402 protocol v2 — exact-scheme USDC transfers, facilitator settles; the facilitator is the fee payer, so the buyer needs USDC only

---

## What the lab is — and what it is not

**What it is:**

- A living lab for the x402 protocol on Solana: tiny paid routes your agent settles with a USDC micro-payment, plus fourteen deterministic failure scenarios (slow serve, verify-time rejection, settle timeouts, facilitator failover, replay protection, expired blockhashes, …) executed on the REAL money path — challenge, verify, settle, serve, receipt.
- Exactly how YOUR x402 client behaves before, during and after settlement: the charged responses, receipts and retry credentials come from the same enforce/settle machinery the production gateway runs — not a mock.
- Fully self-describing: full payment requirements per route at `/.well-known/x402`, an OpenAPI 3.1 spec with live prices, and an agent quick-reference at `/llms.txt`.

**What it is not:**

- **A data service.** The lab sells payment behavior, not data.
- **A key or account requirement:** any generic x402 v2 client works; the payment is the auth.

> **Token analysis (`/check` and related routes) is retired.** Every former analysis surface answers 410 `route_retired` — nothing there is priced or served. What replaced them is the lab; the full guide lives at [/llms.txt](https://svm402.com/llms.txt).

---

## Endpoints & pricing — small amounts, integer USDC base units

Every paid route answers a `402` challenge first; the buyer signs an exact-scheme USDC transfer and the facilitator settles it. Amounts on the wire are integer base-unit strings (`1000` devnet / `2000` mainnet) — never floats. The challenge's `PAYMENT-REQUIRED` header and the `x-x402-price` extension are the only price sources. Scenario walks charge the same lab rate as the ping.

### Paid routes

| Method | Path | Price per call | What it proves |
|--------|------|----------------|----------------|
| GET | `/v1/ping` | `$0.001` devnet / `$0.002` mainnet | The smallest paid call: one request proves your whole x402 v2 round trip — 402 challenge → signed transfer → facilitator verify + settle → 200 whose payment block carries the payer, the settled amount (integer USDC base units), the transaction hash, the explorer link, the facilitator name + fee payer, and verify/settle timings. |
| GET | `/v1/echo` | `$0.001` devnet / `$0.002` mainnet | The echo's GET twin: same price, same settlement; body null and the raw query (`?a=1&b=two`) mirrors exactly. |
| POST | `/v1/echo` | `$0.001` devnet / `$0.002` mainnet | The anything-mirror for client debugging: after the settle it echoes EXACTLY what the gateway received — method, path, raw query, parsed body (up to 16 KB; a bigger body is rejected 413 BEFORE any settle, so an unservable body is never charged), sha256 over the raw bytes, and an allowlisted `headers_seen` where the payment header is redacted and `authorization`/`cookie` never appear. |
| GET | `/v1/scenario/{name}` | `$0.001` devnet / `$0.002` mainnet | Rehearse one deterministic failure walk (the list below) on the REAL verify/settle machinery; the response carries `X-Lab-Scenario: name; step=N` and progress is monotonic per `X-Lab-Client-Id` — "send it again" advances the same ladder. |
| POST | `/v1/scenario/{name}` | `$0.001` devnet / `$0.002` mainnet | The POST spelling of the same walk. |
| GET | `/v1/receipt/{signature}` | `$0.001` devnet / `$0.002` mainnet | The paid on-chain settlement receipt by transaction signature — one `getTransaction` through the lab's shared RPC token bucket, cached by chain finality; a drained budget answers 503 BEFORE any challenge. |

### Free routes — discovery and repair

| Method | Path | Price | What it gives you |
|--------|------|-------|-------------------|
| GET | `/v1/scenarios` | free | The scenario engine's manifest: every walk's name, price, params, mainnet policy and expected status sequence, plus the live lab metrics (`issued` / `redeemed` / `expired` / `charged_unserved`). |
| GET | `/v1/scenario/redeem?client_id=` | free | The recovery lane: serves a previously settled but unserved scenario charge ONCE, free — binding-first (exact path + body fingerprint + payer), single-use; unknown or consumed credentials answer 404, never an error. |
| POST | `/v1/scenario/redeem` | free | The any-method spelling of the recovery lane. |
| GET | `/v1/status` | free | The operational snapshot: each facilitator's probe health (`last_probe_at`, `probe_ms_p50`, the 3-failure breaker), the lab's 24 h settles / charged-unserved / open-credential counters, the fee payer's balance (a background refresher — the route never calls RPC) and the 24 h ping p95. Data older than 90 s answers 503 — never stale-as-fresh. |
| POST | `/v1/diagnose` | free | The payment grader: grade a SIGNED x402 payment (and the challenge it answers) before you send it. 41 offline checks, then 4 ledger checks on at most 2 RPC calls. 200 always — the verdict (`ok \| warnings \| fail`) is data. |
| GET | `/health` | free | The gateway's own liveness. |

---

## The fourteen scenarios — deterministic, on the real money path

Each walk advances per client id; replaying lands on the same step and answers the same verdict, forever. Policy badges reflect how mainnet gates each one. The full walkthroughs — including the payment-header grammar — are in [/llms.txt](https://svm402.com/llms.txt).

- `ok` — verify, settle, 200: the control; every attempt serves. **mainnet: yes**
- `slow` — settle, sleep `?delay_ms=` (0..15000, default 5000), then 200 — the sleep is real, the decision is clock-free. **mainnet: yes**
- `verify_reject` — a 402 with `?reason=` (amount | network | asset | expired); no facilitator call, nothing settles. **mainnet: yes**
- `retryable_503_before_settle` — 503 `{retryable:true}` after verify, BEFORE settle; the next attempt serves. **mainnet: yes**
- `price_change` — a 402 carrying a rotated price (2x devnet; +1000 base units mainnet); paying the new challenge serves. **mainnet: yes**
- `feepayer_rotation` — a 402 whose `extra.feePayer` names the backup facilitator; the re-signed attempt settles via the backup. **mainnet: yes**
- `replay` — the settlement is real exactly once; resending the identical `PAYMENT-SIGNATURE` answers 402 (already used), no second transfer. **mainnet: yes**
- `expired_blockhash` — the challenge carries `extra.recentBlockhash` = a known-expired sentinel; a payment built on it answers 402 `blockhash_expired` with a fresh challenge. **mainnet: yes**
- `backend_error_after_settle` — settle confirmed, then 502 + the durable retry credential; `X-Retry-Settlement` serves free once, reuse falls through to a normal 402. **mainnet: opt-in** — needs `X-Lab-Accept-Fault: 1`
- `settled_503_retry_credential` — settle, then 503 `{status:"settlement_pending", retry_after_s: 2}` WITH the credential; retry free once. **mainnet: opt-in** — needs `X-Lab-Accept-Fault: 1`
- `settle_timeout_then_confirm` — the first settle wait times out client-side while the broadcast proceeds; the late confirm completes the real settle (200 arrives slow). **mainnet: yes**
- `facilitator_down` — the primary is unhealthy for this request only; the backup's challenge is paid and served via the backup. **mainnet: yes**
- `drop_connection_after_settle` — settle, then destroy the TCP socket (devnet only); the credential stays redeemable at `/v1/scenario/redeem`. **devnet only**
- `method_mismatch` — a paid GET answers 405 + `Allow: POST` before any verify; POST rides the money path. **mainnet: yes**

Charged-but-unserved faults (the settled 502 / 503 / drop walks) issue a durable, single-use retry credential — the client is never charged without a way back. The manifest at `/v1/scenarios` publishes the live `charged_unserved` counter.

---

## Devnet quickstart — rehearse before real money

Base URL: `https://lab-devnet.svm402.com` · CAIP-2 network id: `solana:EtWTRABZaYq6iMfeYKouRu166VU2xqa1`.

1. Get devnet SOL for your agent's keypair:

   ```bash
   solana airdrop 2 --url devnet
   ```

2. Get devnet USDC from the [Circle faucet](https://faucet.circle.com) for the Circle devnet mint:

   ```
   4zMMC9srt5Ri5X14GAgXhaHii3GnPAEERYPJgZJDncDU
   ```

3. Call the lab's devnet deployment with an `@x402/svm` client — the 402 → sign → retry loop is automatic on x402 clients, and the facilitator is the fee payer, so you need USDC only:

   ```js
   // Node: @x402/fetch wraps fetch with the x402 v2 payment loop;
   // @x402/svm signs the exact-scheme USDC transfer for Solana.
   import { x402Client, wrapFetchWithPayment } from "@x402/fetch";
   import { ExactSvmScheme, toClientSvmSigner } from "@x402/svm";
   import { createKeyPairSignerFromBytes } from "@solana/kit";
   import bs58 from "bs58";

   // 1. your agent's devnet keypair -> an x402 client signer
   const keypair = await createKeyPairSignerFromBytes(bs58.decode(process.env.AGENT_SECRET_KEY));
   const client = new x402Client();
   client.register("solana:EtWTRABZaYq6iMfeYKouRu166VU2xqa1", new ExactSvmScheme(toClientSvmSigner(keypair)));

   // 2. wrap fetch: 402 -> sign -> automatic retry
   const paidFetch = wrapFetchWithPayment(fetch, client);
   const res = await paidFetch("https://lab-devnet.svm402.com/v1/ping");
   // GET /v1/ping → 402 + PAYMENT-REQUIRED (base64: amount, payTo, asset, fee payer)
   //              → your @x402/svm signer signs the transfer
   //              → automatic retry with the signed envelope → 200 with the
   //                PAYMENT-RESPONSE settlement receipt header
   ```

4. Then run a scenario — same price as the ping, on the same machinery:

   ```js
   const walk = await paidFetch("https://lab-devnet.svm402.com/v1/scenario/ok");
   // every scenario: GET|POST /v1/scenario/{name} — X-Lab-Scenario: name; step=N
   ```

The same client works against mainnet (`https://svm402.com`, CAIP-2 `solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp`) at the mainnet rate (`$0.002` per call) — see the disclaimer below.

---

## Payment flow (x402 v2)

1. Call any paid route. You receive HTTP 402 with a base64-encoded `PAYMENT-REQUIRED` header describing an exact-scheme USDC payment (amount in base units, `payTo`, asset, CAIP-2 network, fee payer). The `payTo` wallet comes from the live challenge — `/.well-known/x402` publishes it per route; nothing to pin here.
2. Sign that transfer with an x402 v2 client — `@x402/fetch` + `@x402/svm`, AgentCash, or any x402 v2 wallet. The facilitator is the fee payer, so the buyer needs USDC only (no SOL for fees).
3. Retry the request with header `PAYMENT-SIGNATURE: <signed payment envelope>`.
4. On success the response carries a `PAYMENT-RESPONSE` receipt header (mirrored as `X-PAYMENT-RESPONSE`) with the settlement hash.

**Charged but not served** (a facilitator or gateway failure after settlement)? Retry the SAME failed request ONCE with header `X-Retry-Settlement: <request id from X-Settlement-Request-Id>` — the gateway serves that request free, without a new payment. The credential is SINGLE-USE and valid only for the FAILED request it was issued for (same path, same body): a request already answered with a 200 is never replayable free. Settled 502/503 responses carry `Retry-After` (seconds) — honor it; the gateway may have already recorded the late settle, and the retry then serves instantly.

---

## Receipts — the settlement, looked up on chain

`GET /v1/receipt/{signature}?pay_to=&amount=&asset=&network=` reports what the chain says about a settle: one `getTransaction`, finality-aware cache (`finalized` and `failed` are cached forever; `not_found` / `confirmed` / `processed` for 10 s). The body reports chain status, slot, block time, fee lamports, the fee payer (labeled), the FIRST USDC transfer matching your `pay_to`, `amount` and `asset` query, and per-check booleans (`pay_to`, `amount_gte`, `asset`, `exactly_one_matching_transfer`); `not_found` is an honest 200. A drained RPC budget answers 503 before any challenge; a budget or RPC failure after your settle answers 503 WITH the live retry credential — you are never charged-and-unserved without a way back.

---

## Diagnose — grade the payment before you send it (free)

`POST /v1/diagnose` grades a SIGNED payment against the 402 challenge it answers: 41 offline checks (envelope ↔ challenge ↔ wire — accepted-copy mismatches including the stale fee payer, the client's ed25519 signature over the exact message bytes, `transferChecked` semantics, compute-budget discipline), then 4 ledger checks on at most two RPC calls. 200 always — the verdict is data, never an HTTP error. A 32 KB raw-byte cap answers 413, and 30 runs/minute/IP answers 429.

The request — a signed payment and the challenge it answers:

```
POST /v1/diagnose
{"payment": "<base64 x402 v2 payment envelope>",
 "challenge": { …the raw 402 body the payment answers… }}
```

A budget refusal or RPC failure is reported as `skipped` with the reason — never a 5xx. The audit line keeps only the payload's sha256; your bytes leave no trace.

---

## Payments — facilitator failover, and whose SOL pays

Challenges settle through **Coinbase CDP** first, then **our own Solana-native facilitator** ([facilitator.svm402.com](https://facilitator.svm402.com)) as backup, then **PayAI** as last resort — automatic failover keeps payments settling if any incumbent stops. Each facilitator's probe health is published at `GET /v1/status`.

**Gas note:** a settle costs ≈ 10,001 lamports — the facilitator pays the SOL fee (it is the fee payer on every challenge); you pay USDC only.

---

## Discovery — point any client at the root

| Surface | Purpose |
|---------|---------|
| `GET /.well-known/x402` | x402 payment discovery metadata — every route with FULL payment requirements (CAIP-2 network, amount, `payTo`, asset, fee payer per resource) |
| `GET /openapi.json` | OpenAPI 3.1 spec with an `x-x402-price` extension on every paid operation |
| `GET /llms.txt` | The agent quick-reference: payment-header grammar, scenario walkthroughs, the payment flow |

Bazaar indexing: MAINNET challenges carry a Bazaar declaration (`extensions.bazaar`, category `Infrastructure`) per paid route; devnet never publishes one — test routes are not indexed.

## AgentCash — any generic x402 client

No per-vendor install needed: any generic x402 v2 client discovers and pays the lab with zero setup.

👉 **AgentCash (recommended): `npx -y agentcash@latest`, then discover `https://svm402.com`** — it lists every route and live price; `agentcash fetch <url>` handles payment automatically.

---

## Disclaimer

This is a PAYMENT lab: every charged call settles a real USDC transfer on the chain the URL serves. Rehearse on devnet first; **mainnet charges are real money**. The lab sells payment behavior, not data, and carries no other warranty.

## Links

- **Service:** https://svm402.com · **Devnet lab:** https://lab-devnet.svm402.com
- **x402 protocol:** https://www.x402.org
- **Client:** any generic x402 v2 client — AgentCash recommended: `npx -y agentcash@latest`