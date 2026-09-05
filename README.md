# openSourceFractal

A Pine Script v6 (TradingView) `indicator` that models a fractal HTF/LTF liquidity setup: **C2 (sweep + reclaim) → CISD confirmation → C3/C4 "T-Spot" equilibrium zones**, with HTF candle visualization, deviation projections, early-warning alerts, and optional SMT divergence confirmation against a correlated instrument.

`max_boxes_count = 500`, `max_lines_count = 500`, `max_labels_count = 500` — generous drawing budgets since a live setup plus history can accumulate boxes/lines/labels quickly.

---

## Core concept

The script watches a higher timeframe (HTF) built from your current chart's lower timeframe (LTF) bars, and looks for a specific liquidity pattern on that HTF:

1. **C2** — the HTF candle sweeps the prior HTF candle's high or low (takes out liquidity) and then closes back inside it (reclaims). This is the first confirmed reversal signal.
2. **CISD** ("Change in State of Delivery") — a specific LTF close level (derived from the swing structure right before the reclaim) that, once broken, confirms delivery has actually flipped direction. Until CISD confirms, a C2 is only a candidate.
3. **C3** — the "T-Spot" equilibrium zone drawn for the leg immediately after CISD confirms (the retracement/entry zone).
4. **C4** — the following HTF leg's T-Spot zone (continuation zone), drawn once the model progresses another period without invalidating.
5. **Early CISD** — a faster, LTF-only version of the reclaim check that can fire *before* the full HTF C2 pattern has technically completed, giving an earlier (but less confirmed) heads-up.
6. **Auto Bias** — an optional higher-timeframe confluence gate: one or two tiers above your primary fractal must *also* be sweeping+reclaiming in the same direction, concurrently, or the primary C2 doesn't trigger at all.
7. **SMT Divergence** — an optional cross-asset check: does a correlated instrument fail to make the same sweep+reclaim on the same leg? If so, that strengthens the case that the move is not broad-based (a classic smart-money-divergence signal).

Everything else in the script (HTF candle drawing, dashboard, deviation lines, history pruning) exists to visualize or manage the lifecycle of these setups.

---

## Input groups

### General Settings
| Input | Effect |
|---|---|
| **Alerts?** | Master switch for all `alert()` calls. If off, nothing fires regardless of other alert toggles. |
| **Early CISD Alerts?** | Sub-switch — only matters if **Alerts?** is also on. Fires when an Early CISD reclaim is detected. |
| **History** | How many past setups (valid or invalidated) stay drawn on the chart. `0` = unlimited; otherwise oldest setups (and their Early CISD line/label records) are deleted once the count is exceeded. |
| **Bias** | `Neutral` / `Bullish` / `Bearish`. Restricts which side of C2 is allowed to form (`allowBuy` / `allowSell`). |
| **Fractal** | The LTF↔HTF pairing. `Auto` derives HTF from your chart's current timeframe (e.g. 1m→15m, 5m→1H, 1H→1D, 1D→1M). Fixed presets (`1m-15m`, `3m-30m`, etc.) lock both sides. `Custom` unlocks the two timeframe pickers below it. |
| **Early CISD** (toggle next to Fractal) | Enables the early/LTF-only CISD detection & drawing pass described above. |
| **Custom LTF / Custom HTF** | Only active when Fractal = `Custom`. Manually choose both timeframes. |

### HTF Candles
Controls the synthetic HTF candle preview drawn to the right of price:
- **HTF Candles** (count, 1–10) and **Hide?** toggle.
- **Offset** — extra horizontal spacing before the candle preview starts.
- **Body / Borders / Wick** colors, separately for bull and bear.
- **HTF Open** — a horizontal reference line at the current HTF candle's open, extending across its width, with its own color/style/width.
- **O/C Time** — vertical lines marking each HTF period boundary, extended both directions.
- **L/H Lines** — horizontal lines tracking the running high/low of each HTF candle as it forms.

### Model Style
Controls the visual language of the setup itself:
- **Labels?** + label size (`Tiny`→`Huge`) — global switch and size for every text label the script draws (C2, C4, CISD, Early CISD, SMT tags, deviation labels).
- **Candle 1 Sweep** — a horizontal line marking the sweep price from the candle that got taken out, drawn between HTF candle midpoints.
- **Early CISD** color/style/width — for the faster LTF-based reclaim lines/labels.
- **Bullish CISD / Bearish CISD** — separate toggle+color/style/width for the confirmed CISD line once a setup fully validates.
- **Candle Equilibrium** — the dotted midline drawn through each HTF candle (and through the C3/C4 T-Spot dash).
- **T-Spot Box** — fill colors (bull/bear) for the C3/C4 equilibrium zone boxes.

### Deviations
- **Enable Projections?** — turns on standard-deviation extension lines drawn from the CISD level once confirmed.
- **Ratios field** (comma-separated, e.g. `-1,-2,-3`) — any set of multipliers; each becomes one projection line + label.
- **Method**: `Wick` (uses the true swing extreme) or `Body` (finds the candle body extreme closest to that same bar) as the basis for the projection range.
- **Color / Style / Width** — shared styling for all projection lines and their numeric labels.

### Auto Bias
- **Auto Bias 1?** — when on, your primary C2 only triggers if a higher-tier fractal is *also* sweeping and reclaiming its own prior HTF level at the same moment. `Auto` steps one tier above your primary fractal automatically; `Custom` lets you lock a specific confirming timeframe instead.
- **Auto Bias 2?** — a second, still-higher confirmation tier stacked on top of Bias 1. If both are enabled, all three tiers (primary + Bias 1 + Bias 2) must agree before a C2 fires.

*Auto tier mapping* (mirrors the same LTF→HTF progression `autoHTF()` already uses for the primary fractal):

| Chart / primary LTF | Primary fractal | Auto Bias 1 | Auto Bias 2 |
|---|---|---|---|
| 1m | 15m | 30m | 1H |
| 3m | 30m | 1H | 4H |
| 5m | 1H | 4H | 1D |
| 15m | 4H | 1D | 1W |
| 1H | 1D | 1W | 1M |
| 4H | 1W | 1M | 1M |
| 1D | 1M | 1M | 1M |

*How it works internally:* two extra `request.security()` feeds (same symbol, higher timeframes) run the identical sweep+reclaim test used for the primary C2 (`c2BuyBias1/2`, `c2SellBias1/2`). Each bias level's *previous*-period result is cached in `PeriodState` the same way the primary condition is, then both are combined into `biasOkBuy` / `biasOkSell`, which directly gates `c2BuyTrigger` / `c2SellTrigger`. This means a disabled Auto Bias level is a no-op (`not enableBias1 or ...` always passes), and an enabled one blocks setup creation entirely rather than just tagging it — no C2, no CISD, no C3/C4, nothing draws until the higher tier(s) agree. When either level is on, its resolved timeframe also shows on the dashboard (`Bias 1 : 30m`, etc.) so you can confirm what's actually being checked.

**Note:** because the mapping is driven by `ltfCode()` (your primary fractal's LTF), it lines up exactly with `Auto`/preset alignments. Manually mismatched `Custom` primary pairings (e.g. an unusually large custom LTF/HTF gap) will still auto-step from the LTF side correctly, but if you want an exact specific bias timeframe regardless of what the auto-stepping would pick, use `Custom` mode on the bias input itself.

### SMT Divergence
- **Enable SMT Divergence?** — master switch.
- **Correlated Asset** — any symbol (`input.symbol`), defaults to `CME_MINI:NQ1!`. Change freely to any correlated futures contract, index, or pair you want to check against.
- **Tag C2 Label?** — whether a confirmed SMT divergence recolors/relabels the C2 label (`C2` → `C2 SMT`).
- **Bullish / Bearish** — the two colors used for SMT-tagged labels.
- **SMT Alerts?** — sub-switch (still gated by the master **Alerts?** toggle) for a dedicated SMT alert, separate from the standard C2 alert.

*How the check works:* on every HTF leg, the script pulls the same open/high/low/close feed from the correlated symbol via a second `request.security()` call and runs the identical sweep+reclaim test used for your primary asset's C2. If your C2 fires but the correlated instrument's equivalent condition does **not** fire on that same leg, the setup is flagged as SMT-divergent. This is a comparative check (did it reclaim or not), not a raw price comparison, so it's valid even between instruments trading at completely different price scales (e.g. ES vs NQ). Because the tag lives on the existing C2 label rather than a separate drawing, an SMT-flagged setup still follows normal C2 lifecycle rules — it still invalidates to `XC2 SMT`, still gets pruned by **History**, etc.

Note: the correlated-asset feed is pulled on the same HTF (`autoTf`) as your primary asset. If the two instruments don't share trading hours (e.g. a future vs a spot FX pair), expect some noise around session opens/closes.

### Dashboard
- **Show Dashboard** toggle.
- **Position** (`Top/Middle/Bottom` × `Left/Center/Right`).
- **Text / Background** colors.
- Displays: title, current LTF-HTF alignment + bias, ticker + HTF close date, and a red warning banner if your chart timeframe doesn't satisfy `validTf` for the selected Fractal setting.

---

## Lifecycle of a setup (internals)

Each side (buy/sell) is tracked as a single `C2Setup` object (`buySetup` / `sellSetup`), replaced whenever a new C2 triggers:

1. **Trigger** — on `periodChanged` (a new HTF bar has started), the script checks whether the *previous* HTF bar satisfied the sweep+reclaim condition (`c2Buy` / `c2Sell`). If so and the bias filter allows it, `newSetup()` builds a fresh `C2Setup`.
2. **Reveal gating** — by design (`requireCisdToShow = true`), nothing is actually drawn until CISD confirms. The setup accumulates state (`c3StartBar`, `c3Top/Bottom`, etc.) invisibly. The moment CISD confirms — whether that happens immediately or several bars/phases later — `reveal()` retroactively draws the C2 label, sweep line, and C3 box/dash backdated to when they actually started (and the C4 box too, if the model had already progressed that far before CISD confirmed).
3. **Phase progression** — `advanceToC4()` closes out the C3 box/dash and opens the C4 (continuation) zone on the next HTF period change, if the setup is still alive.
4. **Invalidation** —
   - `extendAndCheck()`: if price breaches the original C2 extreme, the whole setup dies (`XC2` label, all boxes/lines deleted).
   - `checkC4Invalid()`: if price closes back through the C4 zone, C4 is invalidated (`XC4`). If CISD had *only* ever confirmed while already in C4 (`cisdConfirmedInC4`), the entire setup — including the CISD confirmation itself — is retroactively discarded, since it was never actually valid back at C2/C3.
5. **CISD confirmation** — `checkCisdConfirm()` watches for a close beyond `cisdLevel`; on confirmation it triggers `reveal()` (if not already revealed) and `confirmCisd()`, which draws the CISD line/label and (if enabled) the deviation projections.
6. **History management** — `registerRecord()` pushes every new setup into `setupRecords`; once the count exceeds **History**, the oldest is evicted and its visuals deleted. Early CISD records older than the earliest still-visible setup are pruned the same way.

---

## Alerts

Enable **Alerts?** first, then add a TradingView alert on this indicator (`···` menu → *Add Alert on openSourceFractal*). Individual conditions that can fire (each still gated by **Alerts?**):
- Bullish / Bearish C2 formed
- Early CISD confirmed *(needs "Early CISD Alerts?" on)*
- Bullish / Bearish SMT Divergence *(needs "SMT Alerts?" on)*

All alerts use `alert.freq_once_per_bar`.

---

## Notes / limitations

- `validTf` enforces that your chart's timeframe is at or below the LTF half of the selected Fractal pairing (or, in `Auto` mode, that your current timeframe is one of the pre-mapped ones: 1m/3m/5m/15m/1H/4H/1D). Otherwise the HTF preview is suppressed and the dashboard shows a warning.
- The HTF candle preview and dashboard only draw on `barstate.islast` / `barstate.isrealtime` — they don't repaint historically bar-by-bar, only refresh on the most recent bars.
- Four `request.security()` calls total (primary asset, correlated SMT asset, Auto Bias 1, Auto Bias 2), all with `lookahead = barmerge.lookahead_on` against a higher timeframe — standard, non-repainting usage since the HTF bar being read is always already closed relative to the LTF trigger bar.
- The SMT and Auto Bias calls all run unconditionally regardless of their enable toggles (Pine can't cheaply skip a `request.security()` call based on a boolean input), but this is negligible overhead and well within the 40-call ceiling.
- Auto Bias and SMT are independent: a setup can be SMT-tagged, bias-gated, both, or neither — SMT tags a C2 that already triggered, while Auto Bias controls whether it triggers at all.
