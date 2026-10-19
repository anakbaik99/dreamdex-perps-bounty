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

2. Fetch the integration branch the page tells you to install:

   ```bash
   git clone --depth 1 --branch add-dreamdex-exchange \
     --filter=blob:none --sparse https://github.com/somnia-chain/ccxt
   cd ccxt && git sparse-checkout set ts/src
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

4. Optionally confirm the runtime failure once installed from the branch:

   ```ts
   const ex = new ccxt.dreamdex({ walletAddress: '0x…', privateKey: '0x…' });
   await ex.loadMarkets();
   ex.market('SOMI/USDC');   // BadSymbol
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

So the first line a new integrator copies from this page throws before any network call is made.
The exception text does not name the quote currency, and `USDso` is never mentioned on this page,
so the reader has no local clue where the mistake is. The same defect applies to
`fetchBalance({ symbol })` and all three `vault*` examples, because each takes a market symbol.

The docs contradict themselves on this point. `/developers/http-api/market-data` warns:

> The quote is always USDso. Note the casing and that the native Somnia token is SOMI (not SOM)
> — copy-pasting a placeholder such as SOM:USD returns `market_not_found`.

That warning is on a different page. The CCXT page — the entry point for CCXT integrators, and
the page a reader reaches without having read `market-data` — supplies the placeholder instead.

## Evidence

```
$ grep -c "USDC" ts/src/dreamdex.ts
0
$ grep -c "USDso" ts/src/dreamdex.ts
18
$ python3 -c "import json;d=json.load(open('ts/src/test/static/currencies/dreamdex.json'));print('USDC' in d, list(d))"
False ['SOMI', 'USDso', 'USDC.e', 'WBTC', 'WETH']
$ python3 -c "import json;d=json.load(open('ts/src/test/static/markets/dreamdex.json'));print(list(d))"
['SOMI/USDso', 'WETH/USDso']
```

Branch under test: `somnia-chain/ccxt`, `add-dreamdex-exchange`, commit `d281cb4`.

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

- Network: Somnia Tokyo testnet for markets; the failure occurs before any network call, at
  symbol resolution
- SDK: `ccxt` fork `somnia-chain/ccxt` @ `add-dreamdex-exchange` (`d281cb4`), exchange id
  `dreamdex`
- Docs page: https://docs.dreamdex.io/developers/libraries/ccxt
- Wallet address: not applicable — ccxt rejects the symbol before building a request
