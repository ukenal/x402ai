# Getting indexed in the Coinbase CDP Bazaar

How I got a self-hosted x402 service (three paid routes, Hono on Node, Base mainnet, Coinbase CDP facilitator) indexed in the Coinbase CDP Bazaar, and what I learned keeping it listed. These are seller-side notes from running it, not official documentation.

**Date:** 2026-10-05 · Set up 2026-09-29 and 2026-09-30 on `@x402/*` 2.19.0, re-checked on 2.28.0 · See also the [CHANGELOG](../CHANGELOG.md) and [upgrade notes](upgrading-within-v2.md)

---

## What the Bazaar is

A searchable catalog that the CDP facilitator builds from metadata your service declares in its 402 challenge, once a payment for that route settles through CDP. Per CDP's docs, agents reach it through CDP's REST endpoints (catalog, semantic search, lookup by pay-to address), a Bazaar MCP server, and Amazon Bedrock AgentCore. agentic.market is Coinbase's public storefront over the same catalog, and it also has an editorial, curated tier.

Being indexed makes you findable. It does not make you chosen.

## The two conditions

There is no form and no registration. A route gets indexed after both of these happen:

1. Its 402 challenge advertises `extensions.bazaar`.
2. A payment for that route settles through the CDP facilitator with the resource info (`paymentPayload.resource`) set.

Without the extension, CDP settles your payments and does not index them. That was my situation: my service had settled payments through CDP and still showed zero listings, because none of my routes declared the extension.

Indexing follows CDP-facilitator settlements, so switching to another facilitator would break it.

---

## Wiring

Install `@x402/extensions` at the same version as your other `@x402/*` packages, then declare discovery per route:

```js
import { declareDiscoveryExtension } from '@x402/extensions/bazaar'

const routes = {
  'POST /api/embed': {
    accepts: { scheme: 'exact', price: '$0.01', network: 'eip155:8453', payTo: process.env.WALLET_ADDRESS },
    resource: new URL('/api/embed', baseUrl).toString(),
    description: 'What it does, when to use it, input limits, return fields, typical latency, known limits.',
    mimeType: 'application/json',
    serviceName: 'your-service',
    tags: ['embeddings', 'rag'],
    extensions: declareDiscoveryExtension({
      bodyType: 'json',
      input: { input: 'An example request body.' },
      inputSchema: {
        properties: { input: { type: 'string', minLength: 1, maxLength: 3000, description: 'What this field is.' } },
        required: ['input'],
      },
      output: { example: { embedding: [0.012, -0.034], model: 'your-model' } },
    }),
  },
}
```

Notes from reading the package code:

- `@x402/hono` registers the Bazaar server extension itself when any route declares `extensions.bazaar`, and it validates the declarations when the payment handler is built at startup. There is no manual `registerExtension` step.
- The declared `input` has to validate against your `inputSchema`, or the metadata is rejected.
- `method` is filled in automatically (POST in my case).

## Listing metadata

| Field | Where it goes | Limits |
|---|---|---|
| `description` | Route config, top level | 500 characters maximum. The facilitator rejects longer ones. A bare name scores zero. |
| `serviceName` | Route config, top level | 1 to 32 printable ASCII characters |
| `tags` | Route config, top level | At most 5, each 1 to 32 printable ASCII characters |
| `iconUrl` | Route config, top level | Optional. At most 2048 characters, an absolute http(s) URL with a hostname, no IP literals. I haven't set one. |
| Discovery declaration | `extensions: declareDiscoveryExtension(...)` | Example request, input schema, example output, output schema |

The one that cost me time: **`serviceName`, `tags` and `iconUrl` are siblings of `description` and `mimeType`, not part of `declareDiscoveryExtension`.** I confirmed that by reading `@x402/core` 2.19.0. They are emitted in the challenge's `resource` object and read back from the payment's resource info.

Core validates that `resource` object with a strict parse, so an over-length value can reject the whole payload. The Bazaar extension is more forgiving and just drops invalid fields. Stay inside the limits either way.

For descriptions I say what the route does, when to use it, the input limit, the return fields, typical latency, and known limits ("no web access", "can hallucinate"). Metadata and schemas add to the challenge header: mine is about 2.5 to 2.7 KB per route.

## Unpaid requests must get a 402

Coinbase's free validator (below) POSTs to your paid route without a usable body, and it requires a 402 back. If your service validates input before the payment gate, that probe gets a 400 and fails its `returns_402` check. The fix: an unpaid request always gets 402, whatever the body contains, and input validation (400) runs only when a `payment-signature` header is present. The [migration guide](https://github.com/ukenal/x402-v1-to-v2-migration) covers the middleware pattern.

---

## First settlements

You need one settled CDP payment per route. I paid my own three routes from a separate buyer wallet, which cost $0.07 in total, and the pay-to wallet only receives the funds.

My service logs the extension responses that come back with each settlement. The status values are `success`, `processing`, and `rejected` with a `rejectedReason`. Mine said `processing` every time, and I have never seen `success`; the routes were cataloged anyway. If you see `rejected`, read the reason first. The likely causes are a description over 500 characters or an example input that doesn't match your schema.

A caveat on measuring demand: payments from your own buyer wallet count as calls, and as a payer. If you pay your own routes to get indexed or to refresh, keep those separate from third-party payers when you read the counters.

## Verifying

None of these makes a payment. Replace `YOUR_PAYTO` with your pay-to address, `API` with your base URL, and `YOUR_DOMAIN` with your hostname.

**Check the listing.** This looks up everything indexed under your pay-to address. You should see one resource per route.

```bash
curl -s "https://api.cdp.coinbase.com/platform/v2/x402/discovery/merchant?payTo=YOUR_PAYTO" \
  | jq '{total: .pagination.total, resources: [.resources[]? | .resource // .url]}'
```

**Check the metadata and counters.** This prints each route's service name, tags, description length, and the quality counters (calls in the last 30 days, unique payers, last call time).

```bash
curl -s "https://api.cdp.coinbase.com/platform/v2/x402/discovery/merchant?payTo=YOUR_PAYTO" \
  | jq -r '.resources[] | "\(.resource) name=\(.serviceName|tojson) tags=\(.tags|tojson) desc_len=\(.description|length) quality=\(.quality|tojson)"'
```

**Run Coinbase's pre-flight validator.** It's free and needs no key. Expect `valid: true`, `outcome: accepted` and `failed: 0` for each route. Passing means eligible for indexing, not indexed.

```bash
for r in embed summarize ask; do
  echo -n "$r: "
  curl -s -X POST https://api.cdp.coinbase.com/platform/v2/x402/validate -H "Content-Type: application/json" \
    -d "{\"resource\":\"$API/api/$r\",\"method\":\"POST\"}" \
    | jq -c '{valid, outcome: .simulation.outcome, failed: ([.preflight[] | select(.passed==false)] | length)}'
done
```

**Check what your own 402 advertises.** This decodes the unpaid challenge header and prints whether the Bazaar block is present, plus the service name and tags.

```bash
curl -s -i -X POST "$API/api/embed" -H "Content-Type: application/json" -d '{"input":"check"}' \
  | grep -i '^payment-required' | sed 's/^[Pp]ayment-[Rr]equired: //' | tr -d '\r' | base64 -d \
  | jq '{bazaar: (.extensions|has("bazaar")), name: .resource.serviceName, tags: .resource.tags}'
```

**Check agentic.market's record.** This searches the storefront's public API for your service. Expect your service name and tags once a settlement has gone through. A category and `enriched: true` come only from curation.

```bash
curl -s "https://api.agentic.market/v1/services/search?q=YOUR_DOMAIN" \
  | jq -c '[.services[] | select((.domain // "") | test("YOUR_DOMAIN")) | {domain, serviceName, category, enriched, tags: (.tags // []), endpoints: [.endpoints[].url]}]'
```

## Keeping it listed

A resource with no settlement for 30 days is removed from the catalog and from search, per CDP's docs. Refreshing means one settled payment per route. Metadata changes need three steps: edit the route config, restart, then make one paid call per route.

After a settlement, the new name, tags and descriptions showed up in the merchant lookup within minutes. agentic.market's search picked up the name and tags after that, while its category stayed empty and `enriched` stayed false.

---

## Ranking, and what you don't control

Per CDP's docs, ranking weighs real usage plus listing quality over a rolling 30 days, recomputed every 6 hours. Curation looks for at least 99% availability and is editorial. I found no application path for it. When I looked on 2026-09-30, only 74 of about 2,800 unique services in the agentic.market catalog were `enriched`, which I take to be the curated tier (my inference). Mine is not.

Two things from my own data, each a single snapshot:

- **Descriptions moved rank; tags alone did not.** After I rewrote descriptions and added tags, my embeddings route went from #13 to #3 and my summarize route from #9 to #4 on searches that echoed their descriptions. A bare one-word tag query like "llm" did not surface my routes. Recency and relevance changed together, so I can't separate the two effects. The practical lesson is to write descriptions the way a buyer would phrase the search.
- **Metadata is cheap hygiene, not a growth lever.** In the catalog data I analyzed, traffic was concentrated in a handful of services, and neither tags nor enrichment appeared to gate it. That's my reading of one catalog snapshot.

## Reported issues I haven't reproduced

- [coinbase/cdp-sdk#833](https://github.com/coinbase/cdp-sdk/issues/833): a seller reports that a settled Base mainnet payment never showed up in discovery. The suspected cause is the indexer treating Base as a v1 network and discarding v2 `extensions.bazaar` metadata. It is said to repeat x402 issue #1982. My own routes indexed fine on Base mainnet, so it may not affect every seller.
- [coinbase/cdp-sdk#838](https://github.com/coinbase/cdp-sdk/issues/838): a seller meeting the published curation criteria reports being stuck unlisted on agentic.market for more than 7 weeks.

I'm linking these as reports. I haven't reproduced either.

## Other observations

- **The counters undercount.** I found a one-cent payment from a third-party wallet, made a few days before the upgrade, that the route's payer counter doesn't include. I don't know why. Treat the catalog's payer counts as a lower bound, and cross-check against a block explorer when you want to know who actually paid you.
- **The counters lag.** Payments I made minutes earlier were not in the 30-day counters yet. Check again hours later.
- **`processing` is not a failure.** It was the only verdict I ever got, across every settlement.
- **The validator proves eligibility, not listing.** Always confirm with the merchant lookup.

---

*Written by Landy Ukena from running [x402ai](https://github.com/ukenal/x402ai).*
