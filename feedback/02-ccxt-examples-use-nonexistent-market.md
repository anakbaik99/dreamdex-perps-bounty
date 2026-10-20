# Every CCXT example uses SOMI/USDC, a market and currency that do not exist

Severity: major
Area: docs

## Steps to reproduce

1. Read https://docs.dreamdex.io/developers/libraries/ccxt — the CCXT integration guide. Every
   market argument in every example is `'SOMI/USDC'`:

   ```ts
   await exchange.fetchTicker('SOMI/USDC');
   await exchange.fetchOrderBook('SOMI/USDC');
   await exchange.fetchTrades('SOMI/USDC');
   await exchange.fetchOHLCV('SOMI/USDC', '1h');
   await exchange.createOrder('SOMI/USDC', 'limit', 'buy', 100, 1.5);
   await exchange.watchOrderBook('SOMI/USDC');
   await exchange.vaultApprove('SOMI/USDC', 'SOMI', 100);
   await exchange.vaultDeposit('SOMI/USDC', 'SOMI', 100);
   await exchange.vaultWithdraw('SOMI/USDC', 'SOMI', 50);
   await exchange.fetchBalance({ symbol: 'SOMI/USDC' });
   ```

2. Fetch the integration branch the page tells you to install. Both trees are needed: `ts/src` holds
   the TypeScript source and the fixtures this report reads, `js` holds the built module the runtime
   step imports — checking out `ts/src` alone leaves step 5 with nothing to import:

   ```bash
   git clone --depth 1 --branch add-dreamdex-exchange \
     --filter=blob:none --sparse https://github.com/somnia-chain/ccxt
   cd ccxt && git sparse-checkout set ts/src js
   ```

3. Inspect what the exchange actually supports:

   ```bash
   grep -c "USDC"  ts/src/dreamdex.ts     # 0
   grep -c "USDso" ts/src/dreamdex.ts     # 18

   python3 -c "import json;d=json.load(open('ts/src/test/static/currencies/dreamdex.json'));print('USDC' in d, list(d))"
   # False ['SOMI', 'USDso', 'USDC.e', 'WBTC', 'WETH']

   python3 -c "import json;d=json.load(open('ts/src/test/static/markets/dreamdex.json'));print(list(d))"
   # ['SOMI/USDso', 'WETH/USDso']
   ```

4. Install the one runtime dependency the branch's built module imports (`js/src/base/ws/WsClient.js`
   imports `ws`; without it Node stops at module resolution before any of this report's code runs):

   ```bash
   npm init -y >/dev/null
   npm install ws --no-audit --no-fund
   ```

5. Confirm the runtime failure with the exchange loaded from that branch. No wallet or network is
   needed: seed the client with the branch's own market fixture and call `market()` exactly as the
   docs page does:

   ```js
   const ccxt = (await import('./js/ccxt.js')).default;
   const fixture = (await import('./ts/src/test/static/markets/dreamdex.json',
     { with: { type: 'json' } })).default;

   const ex = new ccxt.dreamdex({});
   ex.setMarkets(fixture);

   ex.market('SOMI/USDC');   // throws
   ex.market('SOMI/USDso');  // resolves
   ```

## Expected

Every example on the page runs against a real market. The quote currency of every dreamDEX
market is `USDso`, so the examples should read `'SOMI/USDso'`.

## Actual

`SOMI/USDC` does not resolve. The exchange has no currency called `USDC` at all:

- `USDC`: 0 occurrences in `ts/src/dreamdex.ts`; `USDso`: 18
- currencies in the branch fixture: `SOMI`, `USDso`, `USDC.e`, `WBTC`, `WETH` — `USDC.e` is a
  bridged token, not the quote of any market, and plain `USDC` is absent
- markets in the branch fixture: `SOMI/USDso`, `WETH/USDso`

CCXT core (`ts/src/base/Exchange.ts`) resolves a symbol by map lookup and has no fallback for an
unknown quote:

```ts
market (symbol: string): MarketInterface {
    if (this.markets === undefined) throw new ExchangeError (this.id + ' markets not loaded');
    if (symbol in this.markets) {
        return this.markets[symbol];
    } else if (symbol in this.markets_by_id) { ... }
    else if ((symbol.endsWith ('-C')) || (symbol.endsWith ('-P')) || ...) {
        return this.createExpiredOptionMarket (symbol);
    }
    throw new BadSymbol (this.id + ' does not have market symbol ' + symbol);
}
```

`SOMI/USDC` is absent from `markets` (keyed `SOMI/USDso`) and from `markets_by_id` (keyed
`SOMI:USDso`) and is not an option symbol, so the call terminates in:

```
BadSymbol: dreamdex does not have market symbol SOMI/USDC
```

So the first line a new integrator copies from this page throws at symbol resolution, before the
ticker request is sent. The exception text does not name the quote currency, and `USDso` is never
mentioned on this page, so the reader has no local clue where the mistake is. The same defect applies
to `fetchBalance({ symbol })` and all three `vault*` examples, because each takes a market symbol.

The docs contradict themselves on this point. `/developers/http-api/market-data` warns:

> The quote is always USDso. Note the casing and that the native Somnia token is SOMI (not SOM)
> — copy-pasting a placeholder such as SOM:USD returns `market_not_found`.

That warning is on a different page. The CCXT page — the entry point for CCXT integrators, and
the page a reader reaches without having read `market-data` — supplies the placeholder instead.

## Evidence

Branch under test: `somnia-chain/ccxt`, branch `add-dreamdex-exchange`, HEAD `d281cb4`
(`fix(dreamdex): use mainnet chain ID 5031, remove type from order responses`).

**Source counts on the integration branch**

```
$ grep -c "USDC" ts/src/dreamdex.ts
0
$ grep -c "USDso" ts/src/dreamdex.ts
18
```

**The branch's own fixtures**

```
$ python3 -c "import json;d=json.load(open('ts/src/test/static/currencies/dreamdex.json'));print('USDC in currencies:', 'USDC' in d, '| keys:', list(d))"
USDC in currencies: False | keys: ['SOMI', 'USDso', 'USDC.e', 'WBTC', 'WETH']

$ python3 -c "import json;d=json.load(open('ts/src/test/static/markets/dreamdex.json'));print('markets:', list(d))"
markets: ['SOMI/USDso', 'WETH/USDso']
```

**Runtime — the documented symbol throws, the corrected one resolves**

Run from an empty directory on 2026-10-20, steps 2 to 5 above verbatim, on branch HEAD `d281cb4`:

```
$ git sparse-checkout set ts/src js    # both trees; js/ccxt.js is 18,472 bytes
$ npm install ws --no-audit --no-fund  # added 1 package
$ node repro.mjs
exchange id            : dreamdex
markets loaded         : SOMI/USDso, WETH/USDso
markets_by_id keys     : SOMI:USDso, WETH:USDso
ex.market('SOMI/USDC') -> BadSymbol: dreamdex does not have market symbol SOMI/USDC
ex.market('SOMI/USDso') -> ok: id=SOMI:USDso base=SOMI quote=USDso type=spot
```

Without `ws` installed the same script stops before any of the above, which is why step 4 exists:

```
Error [ERR_MODULE_NOT_FOUND]: Cannot find package 'ws' imported from
  js/src/base/ws/WsClient.js
```

That is the symbol from the first example on the docs page, resolved by `market()`, the step
`fetchTicker` performs before it builds a request. `BadSymbol` is raised before any request is built,
so the reader gets no network error to investigate — just a symbol error that does not name the quote
currency.

**Live REST API — no `USDC` anywhere**

```
$ curl -s https://stg.api.dreamdex.io/v0/markets | python3 -c "import json,sys; d=json.load(sys.stdin); rows=d if isinstance(d,list) else d.get('markets',[]); print('symbols:', [m.get('symbol') for m in rows])"
symbols: ['WBTC:USDso', 'WETH:USDso', 'SOMI:USDso']

$ curl -s https://stg.api.dreamdex.io/v0/markets | grep -o "USDC" | wc -l
0
```

Every listed market quotes in `USDso`; the string `USDC` does not occur once in the response.

Counts on the docs page itself (rendered, since it is a client-rendered SPA): `SOMI/USDC` appears
10 times, `SOMI/USDso` 0 times.

Checked and correct on the same page, recorded so this report is scoped honestly: the branch
exists and is fetchable; `npm install github:somnia-chain/ccxt#add-dreamdex-exchange` targets a
real ref; the exchange id is `dreamdex` and the class is present in `ts/ccxt.ts`; the claim that
built JavaScript is available is accurate — `js/ccxt.js` (18,472 bytes) and `js/ccxt.d.ts`
(25,432 bytes) both carry the exchange; the supported-method table matches `ts/src/dreamdex.ts`.

Not claimed: `dist/ccxt.browser.js` and `dist/ccxt.cjs` do not contain the exchange while `js/`
does. That may simply be that `dist/` is regenerated at publish time, so it is mentioned only as
an observation.

## Environment

- Network: Somnia Tokyo testnet for markets; the failure occurs at symbol resolution, after
  `loadMarkets()` and before any request is built, so no network is touched by the reproduction
  itself — it seeds markets from the branch's own fixture under `ts/src/test/static/markets/`
- SDK: `ccxt` fork `somnia-chain/ccxt` @ `add-dreamdex-exchange` (`d281cb4`), exchange id
  `dreamdex`
- Docs page: https://docs.dreamdex.io/developers/libraries/ccxt
- Wallet address: not applicable — ccxt rejects the symbol before building a request
