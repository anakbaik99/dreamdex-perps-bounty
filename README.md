# dreamDEX Perps Builder Bounty — feedback submission

Reproducible defects found in the dreamDEX Perps developer surface on Somnia testnet
(chain ID 50312, the environment the app labels `tokyo`).

Each file in [`feedback/`](feedback/) is one item, written to the bounty's required format.
Every item carries the exact request that reproduces it, so it can be checked with `curl`
alone — no account, no token, no screenshot to trust.

## Items

| # | Area | Severity | Title |
|---|---|---|---|
| 01 | docs | normal | [Comma-separated `symbols` returns 400, but the docs promise a silent empty array](feedback/01-symbols-serialization-returns-400.md) |
| 02 | docs | major | [Every CCXT example uses SOMI/USDC, a market and currency that do not exist](feedback/02-ccxt-examples-use-nonexistent-market.md) |

## Quick check

Both API items in #01 reproduce with these three calls and no authentication:

```bash
curl -s -o /dev/null -w '%{http_code}\n' \
  'https://stg.api.dreamdex.io/v0/orderbooks?symbols=SOMI:USDso,WETH:USDso'   # 400, docs say 200 []
curl -s -o /dev/null -w '%{http_code}\n' \
  'https://stg.api.dreamdex.io/v0/tickers?symbols=SOMI:USDso,WETH:USDso'      # 400, docs say 200 []
curl -s -o /dev/null -w '%{http_code}\n' \
  'https://stg.api.dreamdex.io/v0/orderbooks?symbols=SOMI:USDso&symbols=WETH:USDso'  # 200, as documented
```

Item #02 reproduces from the CCXT integration branch named in the docs:

```bash
git clone --depth 1 --branch add-dreamdex-exchange \
  --filter=blob:none --sparse https://github.com/somnia-chain/ccxt
cd ccxt && git sparse-checkout set ts/src
grep -c "USDC"  ts/src/dreamdex.ts    # 0
grep -c "USDso" ts/src/dreamdex.ts    # 18
```

## Scope notes

- Nothing here is reported as a security vulnerability. Security issues would go privately to
  `security@dreamdex.io` and are deliberately absent from this repository.
- All calls are read-only. No order was placed, no state was changed, no wallet was involved.
- Findings that did not survive verification were dropped rather than submitted. The docs were
  also checked in the areas that turned out correct — each item lists what was verified as
  working, so the report is not a one-sided complaint.
