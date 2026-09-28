# VWIN

**A volatility-window indicator for TradingView (Pine Script v6)**

| | |
|---|---|
| **Author** | Subhasom Mandal ([github.com/Subhasom](https://github.com/Subhasom)) |
| **Current release** | 2.1.28092026.1116 |
| **Platform** | TradingView, Pine Script v6, overlay indicator |
| **Product** | **VWIN**: the V-Window on its own, always on: `VWIN.2.1.28092026.1116.pine` |

> ⚠️ **Disclaimer:** VWIN is an analysis tool, not financial advice. It marks periods of high volatility; it does not tell you which way to trade them. Always test on your own instruments and timeframes, and use proper risk management.

---

## Table of contents

1. [What VWIN does](#1-what-vwin-does)
2. [Installation](#2-installation)
3. [Reading the chart](#3-reading-the-chart)
4. [How it works: full logic](#4-how-it-works-full-logic)
5. [Parameter reference](#5-parameter-reference)
6. [How to use it: practical guide](#6-how-to-use-it-practical-guide)
7. [Alerts](#7-alerts)
8. [Limitations and honest notes](#8-limitations-and-honest-notes)
9. [Release versioning](#9-release-versioning)
10. [Credits and licensing](#10-credits-and-licensing)
11. [Implementation notes](#11-implementation-notes)

---

## 1. What VWIN does

VWIN finds the stretches of the chart where volatility expands well beyond its normal level and draws each one as a box. It is the **V-Window** from LCV6 Pro taken out on its own, with everything else removed.

- **Detects volatility bursts.** A window opens when ATR rises above 1.30× its own 50-bar EMA baseline, i.e. when bars are suddenly about a third larger than usual.
- **Follows the move to its end.** When ATR fades, the window is not closed straight away. It stays open until the smoothed trend line (a Nadaraya-Watson kernel regression) turns, so the box covers the whole move the burst started, not just the most volatile bars.
- **Measures what happened.** Each window is drawn as a border-only box from its lowest low to its highest high, with **H** and **L** labels at the extremes and the **% move between them**, e.g. `H/L +2.35%`.
- **Alerts** when a window opens and when it closes.

VWIN is always on: there is no show/hide switch, no summary table, no Buy/Sell signals, no trendlines and no kernel line on the chart. The kernel is still calculated in the background, only to decide when a window closes.

---

## 2. Installation

1. Open [TradingView](https://www.tradingview.com) and any chart.
2. Open the **Pine Editor** tab at the bottom of the screen.
3. Delete the template code, then paste the full contents of `VWIN.2.1.28092026.1116.pine`.
4. Click **Save**, then **Add to chart**.

> 💡 **When updating to a new release**, remove the old indicator from the chart and add the new one. TradingView remembers old settings, so new defaults only apply to a freshly added indicator.

VWIN is light: no lower-timeframe data, no security calls and no history search, so it loads quickly on any chart length.

---

## 3. Reading the chart

| Element | Look | Meaning |
|---|---|---|
| **V-Window box** | Dotted orange border, no fill | One high-volatility window, from the bar it opened to the bar before it closed, and from its lowest low to its highest high. |
| **H label** | Red label pointing down, e.g. `H 6,512.0` | The highest high inside the window, placed on the bar where it was made. |
| **L label** | Teal label pointing up, e.g. `L 6,388.0` | The lowest low inside the window, placed on the bar where it was made. |
| **H/L % move** | Text just outside the box's top-right corner, left-aligned | The % move between the high and the low, in the order they happened. |
| **(Open)** | Added to the % move, e.g. `H/L +1.94% (Open)` | The latest window is still running; the box, labels and % keep updating on the live bar. |

**Reading the % move.**

- **Positive and teal** (`H/L +1.94%`): the low came first, then the high. The burst was a rise, measured from low to high.
- **Negative and red** (`H/L −1.91%`): the high came first, then the low. The burst was a fall, measured from high to low.

The two figures are not mirror images: a rise from 100 to 102 is +2.00%, a fall from 102 to 100 is −1.96%, because each is measured from where the move started.

> **Important:** the high, low and % move are only final once the window closes. They describe what the burst did, for review and context. They are not entry or exit signals.

---

## 4. How it works: full logic

The pipeline runs on every bar:

```
Price data
   │
   ├─► 1. True Range (session-open gaps excluded by default)
   │
   ├─► 2. ATR ÷ its EMA baseline ──► volatility ratio
   │
   ├─► 3. Kernel regression (not drawn) ──► kernel direction
   │
   ├─► 4. Window state: closed → open → extended → closed
   │
   └─► 5. Box, H/L labels and % move ──► alerts
```

### 4.1 Volatility ratio

1. **True Range** is the usual `max(high, previous close) − min(low, previous close)`. With **Count session-open gaps** off (the default), the **first bar of each session** uses `high − low` instead, so an overnight gap cannot open a window by itself.
2. **ATR** is the RMA (Wilder's average) of that True Range over **ATR Length** bars (default 14).
3. **Baseline** is the EMA of the ATR over **Baseline EMA** bars (default 50): what "normal" volatility has recently looked like.
4. **Volatility ratio** = ATR ÷ baseline. A ratio of 1.00 means volatility is at its normal level; 1.30 means it is 30% above it.

### 4.2 Kernel direction (window close only)

A **Nadaraya-Watson estimator with a Rational Quadratic kernel** produces a smooth, low-lag trend line. Each of the last `x + 2` source prices gets a weight:

```
weight(i) = ( 1 + i² / (2 · r · h²) ) ^ (−r)
```

where `h` is the lookback window and `r` the relative weighting. Recent prices get more weight, older ones fade out smoothly.

- By default the kernel counts as **rising (green)** when it is higher than on the previous bar and **falling (red)** otherwise.
- With **Enhance Kernel Smoothing** on, it counts as rising when a faster Gaussian kernel (lookback `h − lag`) is at or above it.
- A **turn up** is a bar where the kernel rises after having fallen on the bar before. This is the event LCV6 marks with a Circle.

The kernel is not drawn. It is identical to LCV6's kernel at the same settings, so VWIN's windows match LCV6 Pro's exactly.

### 4.3 The window cycle

A window moves through three states:

1. **Opens** on the bar where the volatility ratio rises **above** **Open when ATR > baseline ×** (default 1.30). Only one window can be open at a time.
2. **Extends** on the first later bar where the ratio is no longer at or above **Fade below ×** (default 1.05). The window does not close here. VWIN records whether the kernel was **rising** or **falling** at that moment. Once extended, a window stays extended even if volatility picks up again.
3. **Closes** when the kernel turns against the recorded direction:
   - If the kernel was **rising** when ATR faded: after **2 consecutive falling bars**. A one-bar dip is ignored.
   - If the kernel was **falling** when ATR faded: on the first **turn up**.

A new window can open from the very next bar after a close.

### 4.4 Drawing

- On the bar a window opens, VWIN creates the box and its three labels at that bar's high and low.
- While the window is open, it tracks the highest high and lowest low and the bars where they happened.
- When the window closes, the box is fixed from the opening bar to the **bar before the close** (the closing bar is where the kernel turned, which is after the burst), and the labels and % move are set. The closing bar's own high and low are not included.
- A window whose high and low fall on the **same bar** is removed when it closes: it has no move to measure.
- The latest window, while still open, is redrawn on the live bar with " (Open)" added to its % move.

---

## 5. Parameter reference

### V-Window

| Parameter | Default | Description |
|---|---|---|
| Open when ATR > baseline × | 1.30 | Ratio the ATR must exceed to open a window. Higher = fewer, stronger bursts. |
| Fade below × | 1.05 | Once the ratio drops below this, the window switches to its kernel-based extension instead of closing. |
| ATR Length | 14 | Bars in the ATR (Wilder's RMA of True Range). Lower = reacts faster, noisier. |
| Baseline EMA | 50 | Bars in the EMA the ATR is compared with. Higher = steadier idea of "normal". |
| Count session-open gaps | ⬜ Off | On: overnight gaps count in True Range and can open a window by themselves. |
| High / fall | Red | Colour of the H label and of a negative (falling) % move. |
| Low / rise | Teal | Colour of the L label and of a positive (rising) % move. |

### Kernel (window close)

These only affect **when an extended window closes**. The kernel line is not drawn.

| Parameter | Default | Description |
|---|---|---|
| Source | close | Price the kernel is calculated on. |
| Lookback Window (h) | 8 | Bars that carry most of the weight. Lower = turns sooner, so windows close sooner. |
| Relative Weighting (r) | 8 | How quickly older bars lose weight. Lower = longer memory. |
| Regression Level (x) | 25 | Number of bars included in the estimate. |
| Enhance Kernel Smoothing / Lag | ⬜ Off / 2 | Judge the kernel's direction by a crossover with a faster Gaussian kernel instead of by its slope. |

The defaults match LCV6's kernel settings, so windows line up with LCV6 Pro on the same chart.

---

## 6. How to use it: practical guide

**Getting started**

1. Add VWIN to a commodity or intraday chart with default settings.
2. Scroll back through the boxes. Look at how large the % moves are, whether bursts tend to be rises or falls, and where inside each box the high and low were made.
3. Adjust one setting at a time and watch how the number and length of windows change.

**Suggested starting points**

| Situation | Try |
|---|---|
| Too many small windows | Raise **Open when ATR > baseline ×** to 1.40–1.50. |
| Too few windows, missing clear bursts | Lower it to 1.20, or shorten **ATR Length** to 10. |
| Windows run on long after the burst | Lower **Lookback Window (h)** so the kernel turns sooner. |
| Windows close too early, cutting moves short | Raise **Lookback Window (h)**, or turn on **Enhance Kernel Smoothing**. |
| Windows opening right at the session open | Keep **Count session-open gaps** off (the default). |
| 24-hour markets where gaps are part of the move | Try **Count session-open gaps** on. |

**Good practice**

- Use the windows as **context**. They show where the market is moving enough for a trade to cover its costs; they say nothing about direction until the window is over.
- Pair VWIN with your own entry method and treat an open window as "trade here, with care" rather than as a signal.
- Compare the typical % move inside windows with your round-trip costs and stop sizes. If bursts on an instrument are rarely larger than your costs, the instrument is not a good fit.
- Test several months of data before trusting a setting. Thresholds that look best on one stretch of history often fail on the next.

---

## 7. Alerts

Create alerts from TradingView's **Alert** menu → condition **VWIN**:

| Alert | Message | Fires when |
|---|---|---|
| V-Window Open | `VWIN ◧ V-Window Open {{ticker}} {{interval}}` | A V-Window opens |
| V-Window Close | `VWIN ◨ V-Window Close (kernel turned) {{ticker}} {{interval}}` | A V-Window closes |

Each message carries a symbol, ◧ for open and ◨ for close, so the two can be told apart at a glance.

**How `{{interval}}` renders.** TradingView substitutes its own timeframe string: `5`, `15`, `60`, `D`, `W`, `M`, not `5m` or `1h`. `alertcondition()` messages are compile-time constants, so the script cannot reformat them.

Use **"Once Per Bar Close"**. On a still-forming bar the volatility ratio and the kernel can cross their thresholds and cross back, so a window can appear to open or close and then not do so.

> **Important:** a TradingView alert keeps the indicator settings from the moment it was created. If you change a threshold later, delete and recreate the alert.

---

## 8. Limitations and honest notes

- **The window is not a direction signal.** A window opening says volatility has expanded, not which way price will go next. The % move is known only after the window closes.
- **Opening lags the start of the move.** ATR is an average, so it crosses the threshold a few bars into a burst. The box starts on that bar, not at the true beginning of the move.
- **Closing lags the end of the move.** The kernel is smoothed and must turn before a window closes, so windows usually end some bars after the move itself did. That is why the closing bar is left out of the box.
- **The live window changes.** On the last bar the box, H/L labels and % move update as price moves, and a window can open or close intrabar before the bar confirms. Closed windows on history do not change: the script uses no future data and no other timeframe.
- **An extended window cannot re-arm.** Once ATR has faded below the Fade level, a fresh burst inside the same window does not reset it; the window closes on the kernel rule, and a new one can open afterwards.
- **Designed for commodities and intraday charts.** On daily and higher timeframes, or on very quiet instruments, windows can be rare and long.
- **Drawing limits.** TradingView keeps at most 500 boxes and 500 labels per script. Each window uses three labels, so on very long histories the oldest windows lose their labels first (roughly beyond the latest 166 windows), then their boxes.

---

## 9. Release versioning

Releases follow the format:

```
<master release>.<release version>.<DDMMYYYY>.<HHMM>
```

The `DDMMYYYY.HHMM` part is always the date and time (IST) the file was built. It is refreshed on every build, even when the master and release numbers stay the same.

- **Master release** marks a major generation of the script. It changes only when explicitly bumped, and the release version resets to 1 when it does. Currently **2**.
- **Release version** increases by 1 with every code change.
- Date and time are in IST (UTC+5:30).
- File name: `VWIN.<master>.<release>.<DDMMYYYY>.<HHMM>.pine`, e.g. `VWIN.2.1.28092026.1116.pine`. The extension is always lowercase `.pine`.

| Release | Date | Summary |
|---|---|---|
| 2.1.28092026.1116 | 28 Sep 2026 | First release. The V-Window from LCV6 Pro 4.25.28092026.0207 as a standalone indicator, always on. Model, signals, filters, trendlines, kernel line and markers, exits, trade statistics, Liquidity Volume bubbles, Trade within V-Window and the summary table removed. The kernel is kept, undrawn, only for the window-close rule; its settings sit in a new **Kernel (window close)** group. V-Window Open / Close alerts kept, now prefixed "VWIN" and firing whenever a window opens or closes. |

---

## 10. Credits and licensing

The **V-Window** concept and code, its drawing and alerts, and the VWIN build are the work of **Subhasom Mandal**.

---

## 11. Implementation notes

The Pine source carries only short section comments beyond its header; fuller notes live here, in the order the code runs.

**Inputs.** Two groups: **V-Window** (thresholds, ATR and baseline lengths, gap handling, colours) and **Kernel (window close)** (source and the four kernel settings).

**Kernels.** `kernelRQ()` and `kernelGauss()` are the same functions as in LCV6. Their weights depend only on constant inputs, so they and their sum are computed once on the first bar and reused. Both kernels run on every bar, so their state stays consistent and Pine shows no consistency warning; the Gaussian result is only used when **Enhance Kernel Smoothing** is on.

**Volatility ratio.** `ta.tr(true)` returns `high − low` when there is no previous close, so the first bar of the chart is handled. `session.isfirstbar` replaces True Range with `high − low` on each session's first bar unless gaps are counted. The ratio is `na` until the baseline is positive; an `na` ratio never opens a window.

**Window state.** Three persistent flags: `inWin` (a window is open), `winExt` (it has faded and is waiting for the kernel) and `extBear` (the kernel was falling when it faded). `winOpened` and `winClosed` are true only on the bar of the event and drive both the drawing and the alerts. The fade test is written as `not (ratio >= fade)`, so an `na` ratio also counts as faded.

**Drawing.** The box and labels are created when a window opens and are updated only when it closes and on the live bar, not on every bar inside the window, which keeps drawing work low. The right edge is the current bar while open and the bar before the close once closed.

**Alerts.** Two `alertcondition()` calls on `winOpened` and `winClosed`.
