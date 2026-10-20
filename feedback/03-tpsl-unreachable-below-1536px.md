# Set TP/SL is unreachable below ~1536px: the control only exists in the hidden table variant

Severity: normal
Area: app

## Steps to reproduce

Requires an account with at least one open perp position on Somnia Tokyo testnet.

1. Sign in at `https://app.testnet.dreamdex.io` and open any perp market page, e.g.
   `/trade/perps/tokyo/LIT-PERP`.
2. Set the browser window to **1280 × 813** — a common laptop viewport — and reload.
3. In the bottom panel, select the **Positions** tab. The rows render as cards carrying two
   controls each: `Share position` and `Close <PAIR> position`.
4. Query the DOM for the TP/SL control:

   ```js
   [...document.querySelectorAll('[aria-label="Set TP/SL"]')]
     .map(n => { const r = n.getBoundingClientRect(); return `${r.width}x${r.height}`; })
   // ["0x0","0x0","0x0","0x0","0x0","0x0","0x0"]
   ```

   Seven buttons exist, one per open position, all with a zero-size box.

5. Inspect one of them. Its row container is rendered `display: none`:

   ```
   DIV.items-center border-b border-default text-xs hover:bg-surfac … display=none
   ```

6. Resize the window to **1920 × 1080** without navigating. The same seven elements now measure
   `71x32` and the TP/SL action is visible and clickable.

No account funds move at any point; the failure is that the control cannot be reached.

## Expected

`Set TP/SL` is reachable at every supported width. If the narrow layout is intentional, the card
should still expose a way to reach it — a menu, an overflow action, or an entry in the position's
detail view.

## Actual

The app renders **two variants of every position row**:

| viewport | variant | controls present |
|---|---|---|
| ≥ ~1536px | table row | `Set TP/SL`, `Share`, `Close` |
| < ~1536px | card | `Share`, `Close` only |

At the narrower width the table row is still mounted in the DOM but hidden with `display: none`.
The TP/SL button therefore exists, matches its selector, and has no bounding box — so any click
attempt fails with `no bounding box for locator` and the user has no visible control at all.

The breakpoint sits above 1280px, so a reader on a 13" laptop, or with the window not maximised,
cannot set a take-profit or stop-loss from the UI. On a cross-margined venue that is the difference
between a protected and an unprotected position.

## Evidence

Measured on `app.testnet.dreamdex.io`, chain 50383 (Tokyo), 2026-10-19.

**At 1280 × 813** — seven controls, all zero-size:

```
[aria-label="Set TP/SL"]  BOX 0x0  (x7)
hidden by: DIV.items-center border-b border-default text-xs hover:bg-surfac  display=none
```

**Visible controls in the same row at 1280 × 813** — the card variant:

```
Share position: LIT-PERP +2.18%   24x24  (visible)
Close LIT-PERP position           32x32  (visible)
```

**At 1920 × 1080**, same page, no navigation:

```
[aria-label="Set TP/SL"]  BOX 71x32  (x7)
```

**Visible aria-label inventory at 1280 × 813** confirms `Set TP/SL` is absent from the set of
elements with a non-zero box, while `Close … position` is present — the two variants disagree
about which controls they carry.

## Environment

- Network: Somnia Tokyo testnet (chain 50383), app `https://app.testnet.dreamdex.io`
- Region: `tokyo`
- Browser: Chromium 155.0.8059.39, viewport 1280 × 813 and 1920 × 1080
- Account: a smart wallet with 7 open perp positions
- Wallet address: available on request; not required to reproduce, since any account with one open
  position shows the same two variants
