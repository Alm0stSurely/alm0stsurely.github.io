# Trading Analysis — 2026-11-18

## Daily Snapshot

| Metric | Value |
|---|---|
| Portfolio Value | €9,592.59 |
| Daily Change | -38.18 EUR (-0.40%) |
| Total Return (since inception) | -4.07% |
| Cash | €2,077.65 (21.66%) |
| Equal-Weight Benchmark | €9,711.46 (-2.89%) |
| Gap vs Benchmark | -1.19 pp |
| Trades Executed | 0 |

## Session Notes — LLM provider outage (day 2, escalated)

Both the live decision arm and the H4 shadow arm failed today with
`402 Payment Required` from the LLM provider. The pipeline fail-opened to
**hold all positions** — the recorded decision is an error record, not a
deliberate LLM verdict:

> Reasoning: "LLM API error. Holding all positions." (`error: true`, 0 actions)

This is the **second consecutive day** of the outage: on 2026-11-17 only the
shadow arm was affected (live arm produced a normal hold-all decision); today
the live arm's key is rejected as well. No portfolio action was taken or
blocked by this — cash sits at 21.7%, inside the 15–30% band, no
Rule-5 mandate, no stop breaches — but **two consecutive days without a live
decision is now a surveillance item**: if the outage persists, the portfolio
drifts unmanaged through stop/override checks that the mechanical fallback
does not perform. Follow-up needed on provider key/account status.

Shadow arm: error record (`error: true`, empty actions, "LLM API error
(shadow arm).") — no shadow signal today, per the fail-open protocol.

## Risk Metrics

| Metric | Value |
|---|---|
| VaR 95% | +0.72% |
| CVaR 95% | +1.17% |
| Max Drawdown | -2.29% |
| Volatility (ann.) | +6.84% |
| Sharpe | -0.65 |
| Beta (vs SPY) | 0.35 |
| Information Ratio | -1.49 |

Regime: NORMAL vol (adaptive stop 5.0%).

## Open Positions

| Ticker | Qty | Avg Price | Current | Mkt Value | Unreal. P&L | P&L % | Weight |
|---|---|---|---|---|---|---|---|
| SPY | 2.65 | 746.64 | 730.33 | 1,937.06 | -43.26 | -2.18% | 20.2% |
| TLT | 16.31 | 79.97 | 80.22 | 1,308.72 | +4.08 | +0.31% | 13.6% |
| REET | 35.36 | 26.05 | 26.83 | 949.01 | +27.76 | +3.01% | 9.9% |
| DBA | 34.99 | 26.39 | 26.02 | 910.45 | -12.95 | -1.40% | 9.5% |
| TTE.PA | 8.45 | 81.96 | 80.02 | 676.16 | -16.39 | -2.37% | 7.0% |
| MC.PA | 1.15 | 385.40 | 386.95 | 443.98 | +1.78 | +0.40% | 4.6% |
| IJR | 2.91 | 139.02 | 137.88 | 401.71 | -3.32 | -0.82% | 4.2% |
| ^FCHI | 0.05 | 7838.49 | 8148.87 | 383.09 | +14.59 | +3.96% | 4.0% |
| AI.PA | 1.48 | 170.88 | 189.82 | 281.33 | +28.07 | +11.08% | 2.9% |
| PDBC | 12.70 | 17.83 | 17.59 | 223.44 | -3.05 | -1.35% | 2.3% |

## Weekly Summary (W47, through Wednesday)

| Metric | Value |
|---|---|
| Week Start Value (Mon 2026-11-16) | €9,683.42 |
| Current Value | €9,592.59 |
| Week-to-Date | -90.84 EUR (-0.94%) |
| Sessions | 3 (Mon–Wed) |
| Weekly Trades | 2/3 (both Monday Rule-5 re-entries: DBA, TTE.PA) |

## Notes

- No intraday monitor trades today (`trades_history.json` clean for 2026-11-18).
- Largest losers on the day: SPY -2.18% unrealized (still above its entry
  discipline), TTE.PA -2.37% (Monday re-entry, 2 days old).
- AI.PA remains the best performer at +11.08% unrealized.
