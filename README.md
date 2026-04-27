# x402ai

AI microservices monetized via the [x402 payment protocol](https://x402.org) — pay-per-request with on-chain USDC, no accounts, no API keys, no KYC.

Live at **`api.x402ai.dev`** · Base mainnet

---

## Overview

x402ai exposes a set of AI inference endpoints behind a native HTTP 402 payment wall. Each request is individually priced and settled on-chain via EIP-3009 token authorization. The service is self-hosted on a home server and served publicly through a Cloudflare Zero Trust tunnel.

Pricing is fixed per endpoint. Live prices are always available at `GET /health`.

---

## Endpoints

| Method | Path | Model | Price | Latency (warm) | Description |
|--------|------|-------|-------|----------------|-------------|
| `POST` | `/api/embed` | `nomic-embed-text` | $0.01 | ~370ms | Text embedding — 768-dimensional vectors |
| `POST` | `/api/summarize` | `qwen3.5-nothink:2b` | $0.02 | ~8s | Text summarization |
| `POST` | `/api/ask` | `qwen3.5-nothink:2b` | $0.04 | ~7s | Single-turn Q&A |
| `GET` | `/health` | — | free | — | Service status and live pricing |

---

## Payment Flow

x402ai implements the [x402 protocol](https://x402.org) over standard HTTP. No wallet registration or pre-authorization required.

```
Client                          Server                        Facilitator (Coinbase CDP)
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
- The Coinbase CDP facilitator verifies and settles the USDC transfer on-chain
- No session state, no subscriptions, no trust required from the client

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

Payment settlement runs on **Base mainnet** via the Coinbase CDP facilitator.

---

## Stack

| Component | Role |
|-----------|------|
| [Hono](https://hono.dev) | HTTP framework |
| [x402-hono](https://github.com/coinbase/x402) | 402 payment middleware |
| [Ollama](https://ollama.com) | Local model inference runtime |
| `nomic-embed-text` | Text embedding model |
| `qwen3.5-nothink:2b` | Text generation model |
| [Coinbase CDP](https://cdp.coinbase.com) | On-chain payment facilitator |
| [Base](https://base.org) | EVM L2 mainnet (USDC) |
| Cloudflare Zero Trust | Tunnel + edge security |

---

## Why x402?

HTTP 402 ("Payment Required") has been reserved since 1996 but never standardized. The x402 protocol finally makes it practical: a server returns a 402 with machine-readable payment instructions, the client signs an EIP-3009 authorization over the exact amount, and a facilitator settles it on-chain. The server only processes the request after settlement is confirmed.

The result is a fully permissionless API economy — no OAuth, no billing dashboards, no rate-limit tiers. Just sign and pay.

---

## License

MIT
