# x402ai

AI microservices monetized via the [x402 payment protocol](https://x402.org) — pay-per-request with on-chain USDC, no accounts, no API keys, no KYC.

Live at **`api.x402ai.dev`** · Base Sepolia testnet

---

## Overview

x402ai exposes a set of AI inference endpoints behind a native HTTP 402 payment wall. Each request is individually priced and settled on-chain via EIP-3009 token authorization. The service is self-hosted on a home server and served publicly through a Cloudflare Zero Trust tunnel.

Pricing is OCPI-dynamic — pegged to the H100 SXM spot price via the Ornn Compute Price Index and refreshed hourly. If the index is unavailable, the service falls back to hardcoded floor prices.

---

## Endpoints

| Method | Path | Model | Latency (warm) | Description |
|--------|------|-------|----------------|-------------|
| `POST` | `/api/embed` | `nomic-embed-text` | ~370ms | Text embedding — 768-dimensional vectors |
| `POST` | `/api/summarize` | `qwen3.5-nothink:2b` | ~8s | Text summarization |
| `POST` | `/api/ask` | `qwen3.5-nothink:2b` | ~7s | Single-turn Q&A |
| `GET` | `/health` | — | — | Service status and live pricing |

---

## Payment Flow

x402ai implements the [x402 protocol](https://x402.org) over standard HTTP. No wallet registration or pre-authorization required.

```
Client                          Server                        Facilitator (OpenX402)
  │                               │                                    │
  │  POST /api/embed              │                                    │
  │ ─────────────────────────────►│                                    │
  │                               │                                    │
  │  402 Payment Required         │                                    │
  │  X-Payment-Details: { price,  │                                    │
  │    network, recipient, ... }  │                                    │
  │◄─────────────────────────────-│                                    │
  │                               │                                    │
  │  [client signs EIP-3009       │                                    │
  │   transferWithAuthorization]  │                                    │
  │                               │                                    │
  │  POST /api/embed              │                                    │
  │  X-Payment: <signed payload>  │                                    │
  │ ─────────────────────────────►│                                    │
  │                               │  verify + settle on-chain         │
  │                               │ ──────────────────────────────────►│
  │                               │                                    │
  │                               │  settlement confirmed              │
  │                               │◄───────────────────────────────────│
  │                               │                                    │
  │  200 OK + inference result    │                                    │
  │◄──────────────────────────────│                                    │
```

**Key properties:**
- Each request carries its own signed payment authorization
- The OpenX402 facilitator verifies and settles the USDC transfer on-chain
- No session state, no subscriptions, no trust required from the client

---

## Dynamic Pricing (OCPI)

Prices are pegged to H100 SXM spot rates via the **Ornn Compute Price Index** and recalculated hourly. This means inference costs track real GPU market rates rather than being fixed arbitrarily.

```
Ornn Compute Price Index (hourly)
        │
        ▼
  H100 SXM spot rate
        │
        ▼
  per-token cost model  ──►  endpoint price  ──►  402 response
        │
        ▼
  fallback: hardcoded floor prices (if index unreachable)
```

Live pricing is always available at `GET /health`.

---

## Architecture

```
                    ┌─────────────────────────────────────┐
                    │         api.x402ai.dev               │
                    │   Cloudflare Zero Trust Tunnel       │
                    └──────────────────┬──────────────────┘
                                       │ HTTPS
                    ┌──────────────────▼──────────────────┐
                    │         Hono (Node.js)               │
                    │         x402-hono middleware         │
                    │                                      │
                    │   /api/embed  →  Ollama              │
                    │   /api/summarize  →  Ollama          │
                    │   /api/ask  →  Ollama                │
                    │   /health  →  status + pricing       │
                    └──────────────────┬──────────────────┘
                                       │
                    ┌──────────────────▼──────────────────┐
                    │              Ollama                  │
                    │   nomic-embed-text (768-dim)         │
                    │   qwen3.5-nothink:2b                 │
                    └─────────────────────────────────────┘

                    Hardware: Dell Precision, Intel i7-6700
```

Payment settlement runs on **Base Sepolia** (testnet) via the OpenX402 facilitator.

---

## Stack

| Component | Role |
|-----------|------|
| [Hono](https://hono.dev) | HTTP framework |
| [x402-hono](https://github.com/coinbase/x402) | 402 payment middleware |
| [Ollama](https://ollama.com) | Local model inference runtime |
| `nomic-embed-text` | Text embedding model |
| `qwen3.5-nothink:2b` | Text generation model |
| [OpenX402](https://x402.org) | On-chain payment facilitator |
| [Base Sepolia](https://base.org) | EVM testnet (USDC) |
| Cloudflare Zero Trust | Tunnel + edge security |
| Ornn Compute Price Index | Dynamic GPU pricing oracle |

---

## Why x402?

HTTP 402 ("Payment Required") has been reserved since 1996 but never standardized. The x402 protocol finally makes it practical: a server returns a 402 with machine-readable payment instructions, the client signs an EIP-3009 authorization over the exact amount, and a facilitator settles it on-chain. The server only processes the request after settlement is confirmed.

The result is a fully permissionless API economy — no OAuth, no billing dashboards, no rate-limit tiers. Just sign and pay.

---

## License

MIT
