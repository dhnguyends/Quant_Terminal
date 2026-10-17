# Changelog

## 1.2.1 — 2026-10-17

### Fixed: a short trade's exit fee was charged on the wrong amount

Closing a short means buying back what was sold, so the exit fee belongs on
that buy-back amount — for longs it was already right (charged on the sale).
The tester instead charged it on the short's collateral-plus-profit:
`fee × qty × (2×entry − fill)` instead of `fee × qty × fill`.

With fees at 0.1%: shorting at 100 and covering at 50 paid 0.15 per unit
instead of 0.05, so the profit showed 49.75 instead of 49.85 per unit. A
losing short (covering at 150) paid 0.05 instead of 0.15, so the loss looked
0.10 per unit smaller. Winning shorts paid too much, losing shorts too
little; exact only at break-even. Longs were always correct and are
unchanged by this fix. (`public/lib/backtest.js`, `closePos`)

### Tests added

- A winning short and a losing short each pay exactly
  `fee × (entry notional + exit notional)` round trip, pinned to the cent.

## 1.2.0 — 2026-10-16

Two correctness fixes from a full math review of the strategy tester and the indicator library.

### Fixed: trailing stop filled at a price no price path could produce

In the strategy tester's default `intrabar: 'worst'` mode, a trailing stop was ratcheted
to the bar's extreme **before** the stop check, and the exit was then clamped to the bar's
open (`min(stop, open)` for longs, `max` for shorts). When a bar rallied further than the
trail width above its open and then retraced, that clamp filled the trade at the open — a
price that existed *before* the ratcheted stop did. No real price path can produce that
fill, and the error was not bounded by the trail width.

Example (long from 100, 10% trail; bar opens 100, high 112, low 99, close 105): the stop
ratchets to 112 × 0.9 = 100.8 and the low triggers it. The old engine exited at 100 for
zero profit. The up-first truth fills at 100.8 (+0.8%); the down-first truth never triggers
at all. Neither matched the old answer.

Now: a stop that already existed at the open keeps the gap rule (a bar opening beyond it
fills at the open), while a stop ratcheted inside the bar — which did not exist at the
open — fills at its own level, the only price a crossing path guarantees. Mirrored for
shorts. (`public/lib/backtest.js`, `worstExit`)

### Fixed: repainting indicators were allowed as strategy and alert operands

Three indicator outputs are drawn with knowledge of later bars, and could be used directly
in strategy conditions and alerts, so a backtest quietly profited from future data:

- **Ichimoku lagging span** — its value at bar *i* is literally the close at bar *i + 26*
  (`shiftForward(close, -26)`). A rule like `lag.lagging > close` was really asking "will
  price be higher in 26 bars?" and evaluated true on 527 of 1,000 bars of the test fixture
  using future closes.
- **Zig Zag** — a pivot is only confirmed `depth` bars / `deviation`% later and the line
  redraws in between, so a backtest reading the final series "knew" every turn before a
  real trader could. Live alerts could fire on crosses that later un-happened.
- **DPO with Centered = On** — subtracts a moving average centred on the bar, i.e. one
  built from future bars, and shifts the line earlier to match.

Charts are unchanged — all three draw exactly as before. But using them as a strategy or
alert operand is now refused with a plain-language error, the same way marker-only
indicators already are. Backward-looking outputs (Ichimoku's other lines, DPO with
Centered = Off) still work in strategies. (`public/lib/indicators.js` output flags; the
guard lives in `public/lib/backtest.js`, `resolveSeries`.)

### Tests added

- A trailing stop ratcheted inside a bar fills at its level, not the open — long and short.
- Outputs drawn with later bars are refused as operands (Ichimoku lagging span, Zig Zag,
  centered DPO), while the same indicators' backward-looking outputs still resolve.

### Also in this repository

- `1.0.0` — initial import from part-timequant.com, with one local change: the Academy
  promo ribbon is suppressed (`public/js/app.js`, `showRibbon`).
