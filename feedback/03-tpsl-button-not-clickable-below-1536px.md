# The `Set TP/SL` button in the Positions panel cannot be clicked below ~1536px

Severity: normal
Area: app

## Steps to reproduce

Requires an account with at least one open perp position on Somnia Tokyo testnet.

1. Sign in at `https://app.testnet.dreamdex.io` and open any perp market page, e.g.
   `/trade/perps/tokyo/LIT-PERP`.
2. Set the browser window to **1280 × 813** — a common laptop viewport — and reload.
3. In the bottom panel, select the **Positions** tab.
4. Query the DOM for the TP/SL control:

   ```js
   [...document.querySelectorAll('[aria-label="Set TP/SL"]')]
     .map(n => { const r = n.getBoundingClientRect(); return `${r.width}x${r.height}`; })
   // ["0x0","0x0","0x0","0x0","0x0","0x0","0x0"]
   ```

   Seven buttons exist, one per open position, all with a zero-size box — there is nothing to click.

5. Inspect one of them. Its row container is rendered `display: none`:

   ```
   DIV.items-center border-b border-default text-xs hover:bg-surfac … display=none
   ```

6. Resize the window to **1920 × 1080** without navigating. The same seven elements now measure
   `71x32` and the action is visible and clickable.

No account funds move at any point; the failure is that the control has no clickable target.

## Expected

The Positions panel exposes a usable TP/SL action at a 1280px width, the same way it does at 1920px.

## Actual

The app renders **two variants of every position row**:

| viewport | variant | controls present |
|---|---|---|
| ≥ ~1536px | table row | `Set TP/SL`, `Share`, `Close` |
| < ~1536px | card | `Share`, `Close` only |

At the narrower width the table row is still mounted in the DOM but hidden with `display: none`.
The TP/SL button therefore exists, matches its selector, and has no bounding box — a click attempt
fails with `no bounding box for locator`, and the rendered card offers no replacement.

The breakpoint sits above 1280px, so a reader on a 13" laptop, or with the window not maximised,
has no clickable TP/SL control on an open position. On a cross-margined venue that matters: the
position can still be closed, but it cannot be protected from this panel.

## Evidence

Measured on `app.testnet.dreamdex.io`, chain 50383 (Tokyo), 2026-10-19 and re-checked 2026-10-20.

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

### Other paths to TP/SL checked at 1280 × 813

Because "the button is not clickable" is only a defect if there is no other way to reach the same
action, the other entry points were checked on 2026-10-20:

- **Order ticket.** It carries its own protection section: a visible `TP/SL` label (33×18) and two
  checkboxes, in a row whose text reads
  `TP/SL Reduce only TP trigger price Gain % SL trigger price …`. Toggling it did not mount any
  input carrying a trigger-price label — `document.querySelectorAll('input')` returned only the
  order-size field and the two checkboxes. Independently of that, this section applies to the order
  **being placed**; it does not edit an existing position.
- **Position card.** The card variant exposes `Share position` and `Close … position`. Clicking the
  card body opened no dialog in the probe, and no `Set TP/SL` element had a non-zero box.

So within the Positions panel at this width the action is not reachable, and the ticket's TP/SL
section does not substitute for it on an existing position.

## Environment

- Network: Somnia Tokyo testnet (chain 50383), app `https://app.testnet.dreamdex.io`
- Region: `tokyo`
- Browser: Chromium 155.0.8059.39, viewport 1280 × 813 and 1920 × 1080
- Account: a smart wallet with 7 open perp positions at first measurement
- Wallet address: available on request; not required to reproduce, since any account with one open
  position shows the same two variants
