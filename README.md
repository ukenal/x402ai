# x402ai

Self-hosted, payment-gated AI inference microservices on Base mainnet, built on
the x402 protocol. Machines pay machines at the API layer — no accounts, no API
keys, no KYC.

**Live:** https://api.x402ai.dev · https://x402ai.dev
**Registered on x402scan:** https://tryponcho.com/m/api.x402ai.dev

## What it is

Three inference endpoints behind an HTTP 402 payment wall. An agent or developer
sends a signed USDC authorization on Base and gets a result back — no signup
flow, no key management.

| Endpoint | Model | Price |
|---|---|---|
| `POST /api/embed` | nomic-embed-text | $0.01 |
| `POST /api/summarize` | qwen3.5-nothink:2b | $0.02 |
| `POST /api/ask` | qwen3.5-nothink:2b | $0.04 |

Discovery: `/openapi.json` (OpenAPI 3.1) and `/.well-known/x402`.

## Stack

- **Protocol:** x402 V2 (`@x402/hono`, `@x402/core`, `@x402/evm`), header-based
  payment challenge, network `eip155:8453` (Base mainnet)
- **Facilitator:** Coinbase CDP
- **Settlement:** real USDC on Base, `exact` scheme
- **Runtime:** Hono (Node.js) + Ollama for local inference
- **Infra:** self-hosted homelab server, served via Cloudflare Zero Trust tunnel

## Status

- Live on Base mainnet since April 2026
- Migrated to x402 V2 in July 2026 (from the original x402-hono V1 middleware)
- Registered and discoverable on x402scan

The code that runs this service lives on private infrastructure; this repo
documents the architecture and stack for anyone evaluating the work.

x402ai is one piece of a broader line of work in agent-native, machine-to-machine
commerce — building the payment and infrastructure layer that lets autonomous
agents transact directly with services and each other.

## Contact

Landy Ukena — [LinkedIn](https://www.linkedin.com/in/landyukena/)
