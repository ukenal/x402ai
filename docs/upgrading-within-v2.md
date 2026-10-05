# Upgrading within x402 V2: `@x402/*` 2.19.0 → 2.28.0

Notes from upgrading a live, payment-gated API (Hono on Node, Base mainnet, Coinbase CDP facilitator) between two V2 releases, with the method I used to do it safely.

**Date:** 2026-10-05 · **From:** `@x402/core`, `@x402/evm`, `@x402/extensions`, `@x402/hono` 2.19.0 · **To:** 2.28.0 · **Unchanged:** `@coinbase/x402` 2.1.0

If you're migrating from V1 instead, see the [V1 → V2 migration guide](https://github.com/ukenal/x402-v1-to-v2-migration).

---

## Why upgrade

Three upstream releases fixed ways for a request to reach a paid route without paying. All three come down to the same pattern: the payment middleware and the web framework disagreed about which route a path belonged to.

| Version | Fix |
|---|---|
| 2.21.0 | Wildcard route patterns could be bypassed with a percent-encoded line terminator (`%0A`, `%0D`, `%E2%80%A8`). |
| 2.22.0 | A backslash (or `%5C` on Hono) in a `:param` segment could reach a protected handler. |
| 2.27.0 | A literal route such as `/api/premium` could be reached by encoding its path separator (`/api%2Fpremium`). Routes are now matched against both the escaped and the decoded path. |

Sources: the [`@x402/core`](https://github.com/x402-foundation/x402/blob/main/typescript/packages/core/CHANGELOG.md) and [`@x402/hono`](https://github.com/x402-foundation/x402/blob/main/typescript/packages/http/hono/CHANGELOG.md) changelogs.

### My own exposure

Low, and I tested it before upgrading. My paid routes are exact literals (`POST /api/embed` and so on), not wildcards or `:param` routes, and the payment middleware sits behind a path-prefix gate. Before touching anything I sent encoded variants of a paid path to the live service through Cloudflare and again directly to the origin. Neither returned a 200. I upgraded anyway, since the fixes are cheap to take and the protection shouldn't depend on my route layout staying literal.

## Changes to check before you upgrade

I read the changelogs for 2.20.0 to 2.28.0 and compared each entry against my own wiring.

| Version | Change | Why it might matter |
|---|---|---|
| 2.25.0 | `buildPaymentRequirements()` now **throws** when no scheme server is registered for a priced network, instead of warning and returning an empty `accepts` list. Breaking. | Register a scheme for every network you price a route on. |
| 2.25.0 | The process **exits** when the startup facilitator sync fails with a permanent capability or route-configuration error. | A config mistake now fails at boot, not at the first paid request. Under a process manager this can look like a restart loop. |
| 2.22.0 | Unexpected failures return a generic internal error to clients. Every scheme server must declare its payment flows. | Less detail in client-facing errors. The built-in EVM scheme already declares its flows. |
| 2.21.0 | The facilitator client gets a 30 s default per-request timeout, surfaced as a 502. `Cache-Control: no-store` is set on 402 and settlement-failure responses. | Behavior change, not a code change. |
| 2.20.0 | `createAuthHeaders` now throws if it returns a flat headers object instead of one keyed by facilitator path. | Only matters if you build facilitator auth by hand. I use `createFacilitatorConfig` from `@coinbase/x402`. |

Also relevant if you declare Bazaar discovery: `@x402/extensions` 2.21.0 rejects discovery schemas containing external `$ref` or `$id` values.

The 2.28.0 packages pin each other with `~2.28.0` ranges, so move `@x402/core`, `@x402/evm`, `@x402/extensions` and `@x402/hono` together.

---

## Method

The aim is that production stays on the old version until the new one has passed every check I can run for free, then one real payment per route confirms the paid path.

Replace the placeholders with your own: `API` is your public base URL, `ROUTE` is one of your paid paths, and the service directory and process name are yours.

### 1. Baseline: are you exposed today?

This sends an unpaid POST to encoded variants of one paid path and prints the status code for each. `--path-as-is` stops curl from tidying the path. A 402 or 404 is fine. A **200** means the request reached your handler without paying.

```bash
API=https://your-api.example
for p in "/api%2Fembed" "/api/embed%0d%0a" "/api/embed%0a" "/api%5Cembed" "/api/embed/" "/api//embed" "/api/%65mbed" "/api%2fembed"; do
  echo -n "$p -> "
  curl -s -o /dev/null -w "%{http_code}\n" -X POST --path-as-is "$API$p" \
    -H "Content-Type: application/json" -d '{"input":"check"}'
done
```

Run it again against the origin (for example `http://localhost:3000`) to find out whether your CDN or tunnel is doing any of the blocking.

My results: `%2F`, `%2f`, `%5C`, `%0a` and `%0d%0a` returned 404. The trailing-slash, double-slash and `%65` (an encoded letter) variants returned 402, meaning the server normalizes them to the real route and the payment gate still catches them. The origin gave the same results as the public endpoint.

### 2. Tag a rollback point

```bash
git status --short && git tag pre-x402-2.28 && git ls-files package.json package-lock.json
```

Confirms a clean tree, tags the current commit, and confirms both package files are tracked, which is what makes a rollback possible.

### 3. Stage a copy

This copies the service to a sibling directory, drops the copy's `.git` so you can't commit from it by mistake, and switches its listen port so it can't collide with production. Adjust the port edit to however your service sets its port. The copy contains your `.env`, so delete it when you're done.

```bash
cp -a ~/your-service ~/your-service-stage && rm -rf ~/your-service-stage/.git
sed -i 's/port: 3000/port: 3100/' ~/your-service-stage/index.js
```

### 4. Upgrade the copy

`--save-exact` writes exact pins. `--legacy-peer-deps` skips the unused paywall peer that wants React 19. Drop `@x402/extensions` if you don't use it.

```bash
cd ~/your-service-stage && npm install --legacy-peer-deps --save-exact \
  @x402/core@2.28.0 @x402/evm@2.28.0 @x402/extensions@2.28.0 @x402/hono@2.28.0
npm ls @x402/core @x402/hono @x402/evm @x402/extensions @coinbase/x402 --legacy-peer-deps
```

Every `@x402/core` entry should read 2.28.0 or `deduped`. A second copy of core at the old version nested under `@coinbase/x402` would be the thing to look for. I didn't get one.

### 5. Boot it and test

Start the copy in the background and confirm the port before testing anything, so you know you're not testing production:

```bash
cd ~/your-service-stage && (node index.js > /tmp/stage.log 2>&1 & echo $! > /tmp/stage.pid)
sleep 8 && cat /tmp/stage.log && ss -ltnp | grep -E ':3000|:3100'
```

Then three free checks, none of which signs or pays anything:

**Compare the 402 challenge, old against new.** This fetches the unpaid challenge from production and from the staged copy, decodes the base64 `payment-required` header, sorts the keys, and diffs the two. It prints `IDENTICAL` if nothing a buyer or a discovery crawler can see has changed.

```bash
ch() { curl -s -i -X POST "http://localhost:$1/api/embed" -H "Content-Type: application/json" -d '{"input":"check"}' \
  | grep -i '^payment-required' | sed 's/^[Pp]ayment-[Rr]equired: //' | tr -d '\r' | base64 -d | jq -S .; }
diff <(ch 3000) <(ch 3100) && echo IDENTICAL
```

**Repeat the path-variant loop** from step 1 against port 3100.

**Confirm unpaid requests still get 402**, including with no body at all, on every paid route:

```bash
for r in embed summarize ask; do echo -n "$r: "; curl -s -o /dev/null -w "%{http_code}\n" -X POST http://localhost:3100/api/$r; done
```

My results: the decoded challenge was byte-identical to production, the path variants matched the baseline exactly, and all three routes returned 402.

Stop the copy when you're satisfied: `kill $(cat /tmp/stage.pid)`.

### 6. Promote

Install the same pins in production, then restart. `pm2 flush` clears old log scrollback first so what you read afterwards is fresh. Skip the pm2 parts if you use something else.

```bash
cd ~/your-service && npm install --legacy-peer-deps --save-exact \
  @x402/core@2.28.0 @x402/evm@2.28.0 @x402/extensions@2.28.0 @x402/hono@2.28.0
git --no-pager diff package.json
pm2 flush your-service && pm2 restart your-service && sleep 8 && pm2 logs your-service --nostream --lines 30
```

The `package.json` diff should show only the four version lines. The logs should show a clean start and an empty error log.

### 7. Check the public side

Through your CDN or tunnel, confirm unpaid requests still get 402 and, if you list in the Coinbase CDP Bazaar, that Coinbase's free validator and the merchant lookup still pass. Replace `YOUR_PAYTO` with your payment address.

```bash
for r in embed summarize ask; do
  echo -n "$r public: "; curl -s -o /dev/null -w "%{http_code}  " -X POST "$API/api/$r"
  curl -s -X POST https://api.cdp.coinbase.com/platform/v2/x402/validate -H "Content-Type: application/json" \
    -d "{\"resource\":\"$API/api/$r\",\"method\":\"POST\"}" \
    | jq -c '{valid, outcome: .simulation.outcome, failed: ([.preflight[] | select(.passed==false)] | length)}'
done
curl -s "https://api.cdp.coinbase.com/platform/v2/x402/discovery/merchant?payTo=YOUR_PAYTO" | jq '{total: .pagination.total}'
```

I expected 402 and `valid: true, accepted, failed: 0` three times, and a merchant total of 3. I got exactly that.

### 8. One real payment per route

The free checks can't tell you whether verify and settle work on the new version, so make one small paid call per route with your usual test client. Mine settled on-chain on all three. Timings were in line with earlier runs (about 50 ms for embeddings, 6 s for summarize, 9 s for ask on CPU-only inference), so the upgrade added no visible latency.

If you use the Bazaar, read the settlement verdict in your logs. Mine was `processing` on every payment, as before.

### 9. Commit and clean up

```bash
git add package.json package-lock.json && git commit -m "Upgrade @x402/* to exact 2.28.0"
rm -rf ~/your-service-stage /tmp/stage.log
```

Commit only the two package files, then delete the staged copy because it holds a copy of your secrets.

---

## Rollback

Restores the two package files from the tag, reinstalls exactly what the lockfile says, and restarts:

```bash
git checkout pre-x402-2.28 -- package.json package-lock.json && npm ci --legacy-peer-deps && pm2 restart your-service
```

## Observations

- **Two extension-response lines per payment.** After the upgrade my logs show the Bazaar extension response twice per payment, once before the handler runs and once after, where I used to see one. I believe it comes from the 2.25.0 change that populates extension responses on verify as well as settle, but I haven't confirmed that in the source. It matters only if you parse those log lines and expect one per payment.
- **Startup failures.** Because 2.25.0 can exit the process on a permanent sync error, a startup promise without a `.catch` will crash with a bare unhandled rejection. Mine currently has none. Adding one that logs a clear message is a small separate change.
- **What this didn't cover.** I did not read the 2.28.0 source for validation changes to route-level fields such as `serviceName` and `tags`. After the upgrade my listing name, tags and descriptions were intact in the Bazaar catalog, which is the check that matters in practice.

---

*Written by Landy Ukena from the upgrade of [x402ai](https://github.com/ukenal/x402ai).*
