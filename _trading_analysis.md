# Trading Analysis — 2026-11-19

## Daily Snapshot

| Metric | Value |
|---|---|
| Portfolio Value | €9,534.24 |
| Daily Change | -58.35 EUR (-0.61%) |
| Total Return (since inception) | -4.66% |
| Cash | €2,077.65 (21.79%) |
| Equal-Weight Benchmark | €9,654.97 (-3.45%) |
| Gap vs Benchmark | -1.21 pp |
| Trades Executed | 0 |

## Session Notes — LLM provider outage (day 3, pipeline now in ALERT state)

Both the live decision arm and the H4 shadow arm failed again today with
`402 Payment Required` from the LLM provider. The pipeline fail-opened to
**hold all positions** for a third straight session:

> Reasoning: "LLM API error. Holding all positions." (`error: true`, 0 actions)

The pipeline's own outage tracker has now crossed its threshold:
`results/llm_outage_status.json` reports `state: ALERT` (2 consecutive error
days tracked since 2026-11-18; in the wild this is day 3 — the shadow arm
already failed alone on 2026-11-17).

This is no longer a market story, it is the **primary pipeline risk**. The
fail-open fallback performs no stop checks, no regime checks and no cash-band
(Rule-5) logic — and the market is quietly drifting against the book:

- SPY is now **-4.13% unrealized** (€1,898.53, 19.9% of the portfolio) —
  roughly **0.9 pp from the 5% adaptive stop** in the current NORMAL-vol
  regime. Nobody is evaluating that distance. The intraday monitor's
  mechanical stop alerts are the only line of defense still live (it ran
  twice today and stayed silent after one early alert — see below), but the
  evening pipeline itself would not act on a stop breach while the provider
  is down.
- TTE.PA (-3.14%) and IJR (-3.17%) are also within ~2 pp of the stop.

If a fourth consecutive error day occurs, an explicit exit-rule review is
warranted rather than letting the exception silently become the rule. The
stop-override policy, the deferred-exit framework, and the weekly-budget
arithmetic all assume a functioning decision arm.

Shadow arm: error record again (`error: true`, empty actions, "LLM API error
(shadow arm).") — no shadow signal today. The live–shadow comparison
apparatus is itself blind until the provider is restored.

**Intraday monitor**: 2 runs, 1 alert, 0 trades. At 12:15 UTC a
`POSITION_MOVEMENT` alert fired on **MC.PA** (+2.11% vs the stale pipeline
reference of €386.95; real P&L +2.50% at the time). The alert session held:
RSI ~53, BB position 0.30, no stop, no band break — "cooldown expired"
surveillance after the recent pullback, not a breakout. By 17:45 UTC the
run was fully silent; MC.PA closed the Paris session at **€391.60**
(+1.61% unrealized vs avg €385.40), never approaching the +5% escalation
zone or the upper band (~€418.7). The last weekly budget slot (1/3) carries
to Friday.

## Risk Metrics

| Metric | Value |
|---|---|
| VaR 95% | +0.75% |
| CVaR 95% | +1.18% |
| Max Drawdown | -2.87% |
| Volatility (ann.) | +6.91% |
| Sharpe | -1.84 |
| Beta (vs SPY) | 0.35 |
| Information Ratio | -1.49 |

Regime: NORMAL vol (adaptive stop 5.0%). Return distribution mildly
left-skewed (skew -0.28, excess kurtosis +1.04).

## Open Positions

| Ticker | Qty | Avg Price | Current | Mkt Value | Unreal. P&L | P&L % | Weight |
|---|---|---|---|---|---|---|---|
| SPY | 2.65 | 746.64 | 715.80 | 1,898.53 | -81.79 | -4.13% | 19.9% |
| TLT | 16.31 | 79.97 | 80.35 | 1,310.76 | +6.12 | +0.47% | 13.7% |
| REET | 35.36 | 26.05 | 26.38 | 932.92 | +11.67 | +1.27% | 9.8% |
| DBA | 34.99 | 26.39 | 26.22 | 917.28 | -6.12 | -0.66% | 9.6% |
| TTE.PA | 8.45 | 81.96 | 79.39 | 670.83 | -21.72 | -3.14% | 7.0% |
| MC.PA | 1.15 | 385.40 | 391.60 | 449.31 | +7.11 | +1.61% | 4.7% |
| IJR | 2.91 | 139.02 | 134.62 | 392.21 | -12.82 | -3.17% | 4.1% |
| ^FCHI | 0.05 | 7838.49 | 8084.91 | 380.08 | +11.58 | +3.14% | 4.0% |
| AI.PA | 1.48 | 170.88 | 189.28 | 280.53 | +27.27 | +10.77% | 2.9% |
| PDBC | 12.70 | 17.83 | 17.65 | 224.13 | -2.35 | -1.04% | 2.3% |

## Weekly Summary (W47, through Thursday)

| Metric | Value |
|---|---|
| Week Start Value (Mon 2026-11-16) | €9,683.42 |
| Current Value | €9,534.24 |
| Week-to-Date | -149.18 EUR (-1.54%) |
| Sessions | 4 (Mon–Thu) |
| Weekly Trades | 2/3 (both Monday Rule-5 re-entries: DBA, TTE.PA) |

## Notes

- No intraday monitor trades today (`trades_history.json` clean for 2026-11-19);
  the only intraday event was the MC.PA alert described above.
- Two consecutive down days while unmanaged: -0.40% (11-18), -0.61% (11-19).
  The equal-weight benchmark is falling too (-3.45% since inception), so the
  gap has been stable around -1.2 pp — small comfort.
- AI.PA remains the standout at +10.77% unrealized; worst book is SPY
  -4.13%, now the closest position to its adaptive stop.
- Cash at 21.79% is comfortably inside the 15–30% band — no Rule-5 mandate,
  and even if there were one, the fallback could not execute it.
- Friday is the last W47 session with 1/3 weekly budget remaining; whether
  the provider is back by then determines if Friday is a real decision.

## Research Session Notes (2026-11-19, post-close)

- **Shadow-arm visibility shipped.** The daily JSON now carries an
  informational `shadow_arm` block next to `llm_outage` — a
  shadow-arm-only outage (like 2026-11-17) is now visible without opening
  the shadow log. The block is deliberately fenced off from the live-arm
  escalation streak: a broken experiment must never escalate pipeline
  state. Five new regression tests pin that invariant (suite: 1831).
- **H4 shadow gate check:** n=6/8, mean 5D forward return −0.58%,
  positive 2/6 (33%). Still well under the pre-registered bar. DBA and
  TTE.PA (11-16 entries) remain pending; the window extends, the bar
  does not move.
- **Cash-alpha correlation:** re-confirmed null (Pearson r = +0.019,
  n=102). The alpha gap is a level effect — chronic under-investment vs
  a fully-invested benchmark — not a daily timing signal. Cash-drag
  diagnosis: 64 drag days vs 22 cap-binding days; the prompt, not the
  cap, is the lever. (No prompt change while H4 is in shadow.)
- **Keyword trends:** `cash band` 100% latest week, +3.41 pp/wk rising;
  `let winners run` 50%, +0.97 rising. `deflated sharpe` 50% latest
  but −2.03 pp/wk falling; the H-watch 2-week window closes ~2026-11-20
  with no organic re-adoption.
- **Escalation policy reminder:** Venice 402 is at day 3. If day 4
  occurs tomorrow, the stated policy is an explicit exit-rule review —
  SPY sits 0.87 pp from its 5% adaptive stop with no live decision layer
  to react.
