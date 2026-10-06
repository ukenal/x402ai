# Changelog

Notable changes to the live x402ai service (<https://api.x402ai.dev>). Dates are when the change went live. Entries link to more detail where it exists.

## 2026-10-05

- Upgraded `@x402/core`, `@x402/evm`, `@x402/extensions` and `@x402/hono` from 2.19.0 to 2.28.0 (exact pins). Picks up three upstream fixes to how paid routes are matched against encoded request paths. No change to endpoints, prices or listing metadata. Staged first, then verified with one paid call per route. Details: [docs/upgrading-within-v2.md](docs/upgrading-within-v2.md)

## 2026-09-30

- Added listing metadata on all three routes: service name, tags, rewritten descriptions, and input/output schema descriptions and examples.

## 2026-09-29

- Enabled the Bazaar discovery extension on all three routes; the routes were indexed in the Coinbase CDP Bazaar after settled payments.
- Unpaid requests now always return 402. Input validation (400) runs only when a payment signature is present.

## 2026-07-22

- Migrated from x402 V1 to V2, registered on x402scan, and unified the request field to `input`. Details: [x402-v1-to-v2-migration](https://github.com/ukenal/x402-v1-to-v2-migration)

## 2026-04-27

- Moved to Base mainnet with the Coinbase CDP facilitator. Prices set at $0.01 (embed), $0.02 (summarize) and $0.04 (ask).
