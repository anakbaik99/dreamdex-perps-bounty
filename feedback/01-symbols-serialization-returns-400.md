# Comma-separated `symbols` returns 400, but the docs promise a silent empty array

Severity: normal
Area: docs

## Steps to reproduce

1. No authentication needed. Send a comma-separated `symbols` value to either array-valued
   endpoint the docs name:

   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' \
     'https://stg.api.dreamdex.io/v0/orderbooks?symbols=SOMI:USDso,WETH:USDso'

   curl -s -o /dev/null -w '%{http_code}\n' \
     'https://stg.api.dreamdex.io/v0/tickers?symbols=SOMI:USDso,WETH:USDso'
   ```

2. Compare against the control that does work as documented:

   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' \
     'https://stg.api.dreamdex.io/v0/orderbooks?symbols=SOMI:USDso&symbols=WETH:USDso'
   ```

3. Read the docs page under test:
   https://docs.dreamdex.io/developers/http-api/market-data — section "Order book depth".

## Expected

Per the docs, the comma-separated form is parsed as one literal symbol, matches no market and
is **silently skipped**, so the response is `[]`. The page states this twice, in prose and as a
runnable example:

> Wrong — parsed as one literal symbol "SOMI:USDso,WETH:USDso", which matches no market and is
> **silently skipped, so the response is []**

> Unknown or malformed symbols are **silently skipped rather than erroring**, so a serialization
> mismatch shows up as an **empty [] rather than a 4xx**.

Expected: HTTP `200` with an empty array on both endpoints.

## Actual

HTTP `400` with a machine-readable error name and a structured context object:

```json
{
  "context": { "invalid_symbols": ["SOMI:USDso,WETH:USDso"] },
  "description": "unknown market symbols: SOMI:USDso,WETH:USDso",
  "name": "invalid_param",
  "status": 400
}
```

Observed status codes, in the order of step 1 and step 2 above:

```
400   /v0/orderbooks?symbols=SOMI:USDso,WETH:USDso
400   /v0/tickers?symbols=SOMI:USDso,WETH:USDso
200   /v0/orderbooks?symbols=SOMI:USDso&symbols=WETH:USDso
```

Full matrix, all read-only, no auth:

| Query on `/v0/orderbooks` | Status | Documented expectation |
|---|---|---|
| `?symbols=SOMI:USDso,WETH:USDso` | 400 `invalid_param` | `[]` (silently skipped) |
| `?symbols=WETH:USDso,SOMI:USDso` | 400 `invalid_param` | `[]` (silently skipped) |
| `?symbols=NOPE:USDso` | 400 `invalid_param` | `[]` (silently skipped) |
| `?symbols=` | 400 `invalid_param` ("empty value is not allowed") | not documented |
| `?symbols=SOMI:USDso&symbols=WETH:USDso` | 200, 2 items | 200, 2 items |
| `?symbols=SOMI:USDso` | 200, 1 item | 200, 1 item |

`/v0/tickers` behaves identically on every error row.

Impact: the page tells the integrator not to expect the class of error that actually occurs
("rather than a 4xx"), and prescribes the wrong diagnosis — an empty response means "the market
has no book", when the real fault is the caller's serialisation. A bot written from this page
treats the 400 as an unhandled crash.

Secondary on the same surface: `/developers/http-api/error-handling` documents three response
fields (`status`, `name`, `description`). Live error responses also carry `context`, which is
the only place the offending values are returned — so a useful error message cannot be built
from the documented fields alone.

## Evidence

```
$ curl -s -o /dev/null -w '%{http_code}\n' 'https://stg.api.dreamdex.io/v0/orderbooks?symbols=SOMI:USDso,WETH:USDso'
400
$ curl -s -o /dev/null -w '%{http_code}\n' 'https://stg.api.dreamdex.io/v0/tickers?symbols=SOMI:USDso,WETH:USDso'
400
$ curl -s -o /dev/null -w '%{http_code}\n' 'https://stg.api.dreamdex.io/v0/orderbooks?symbols=SOMI:USDso&symbols=WETH:USDso'
200
```

Full response body for the first request:

```json
{"context":{"invalid_symbols":["SOMI:USDso,WETH:USDso"]},"description":"unknown market symbols: SOMI:USDso,WETH:USDso","name":"invalid_param","status":400}
```

Checked and correct, recorded so this report is scoped honestly: the `/v0` prefix is required
and its absence really does 404; the hard-coded testnet symbol list matches `/v0/markets` and
`/v0/currencies` exactly; both documented "these 404" paths do return 404; repeated-key
serialisation works as documented; `GET /v0/auth/nonce` returns a nonce.

## Environment

- Network: Somnia testnet, chain ID 50312 (`0xc488` confirmed via `eth_chainId`); the app labels
  this environment `tokyo` in its trade URLs (`/trade/perps/tokyo/...`) and the docs call it
  Somnia Shannon testnet
- REST base: `https://stg.api.dreamdex.io/v0`
- No SDK involved — direct HTTP, no auth token required for these endpoints
- Wallet address: not applicable (read-only endpoints, no signature or session)
