# Cross-Sectional Equity Alpha Research

A point-in-time backtesting framework for US large-cap equities, with realistic
transaction costs and out-of-sample validation.

**TL;DR** — A beta-neutral composite signal returned **+2.6%/yr at a Sharpe of
0.55** in development (2009–2023) and **−6.7%/yr at a Sharpe of −0.86** on a
32-month holdout sealed before any tuning. **The strategy did not survive out
of sample.**

That is the headline result, and it is consistent with the project's main
finding rather than a surprise: the signal's alpha decayed monotonically from
+2.8%/yr (2010–14) to +0.2% (2020–23), and the holdout is simply the next point
on that curve. Along the way the project found and corrected **seven
measurement biases** in its own results, two of which were inflating Sharpe by
1.8×.

---

## Significance

Most public backtests report a Sharpe ratio and stop. The interesting content
here is the machinery that decides whether a Sharpe ratio means anything:

- **Point-in-time universe** reconstructed from historical index membership, so
  companies that failed are present in the data rather than silently dropped
- **Embargoed walk-forward validation** — the target is a 21-day *forward*
  return, so training data is cut off 21 sessions before each test date
- **Square-root market impact** scaled by participation in each name's own
  average daily volume, plus spread and short borrow charged as a holding cost
- **Deflated Sharpe ratios** that adjust for the number of variants searched
- **A locked holdout** of 32 months, sealed before any tuning

And a research log that records the ideas that **failed**, which is most of
them.

---

## The holdout

32 months (2024–2026) were sealed before any tuning and evaluated once, at the
end. Development and holdout are measured at the same fund size.

| Strategy | Dev Sharpe | Holdout Sharpe | Dev IC | Holdout IC |
|---|---:|---:|---:|---:|
| **Composite, beta-neutral** | **+0.55** | **−0.86** | +0.012 | **−0.040** |
| ML long/short | −1.11 | −1.48 | −0.001 | −0.027 |
| Time-series momentum | −0.48 | +0.85 | — | — |

The composite's sign flipped. Because the composite has **no fitted
parameters** — it is a fixed rank-average with signs taken from published
literature — this is not overfitting in the usual sense. There was nothing to
overfit. It means the underlying **effect stopped working**, which is what the
decay curve predicted.

The momentum book flipping positive on 32 months is not evidence of anything;
it was negative across 228 development months, and a 32-month window at these
effect sizes has very wide error bars.

## Development results

These are the in-sample numbers. **Read them in light of the holdout above.**

All figures are net of modelled costs, measured over 180 walk-forward months
(2009–2023) on an identical window for every strategy.

| Strategy | Return | Sharpe | 95% CI | Beta | Alpha | Alpha *t* |
|---|---:|---:|---|---:|---:|---:|
| **Beta-neutral composite ($25M)** | **+2.6%** | **0.55** | [0.11, 0.98] | 0.03 | **+2.1%** | 1.72 |
| Composite, long-tilted | +15.8% | 1.06 | [0.61, 1.64] | 0.91 | +0.7% | 0.61 |
| ML long-only leg | +14.9% | 0.93 | [0.49, 1.48] | 1.01 | −1.6% | −1.51 |
| ML long/short | −5.1% | −1.10 | [−1.55, −0.69] | 0.01 | −5.2% | −4.35 |
| *SPY (buy and hold)* | *+16.6%* | *1.06* | *[0.63, 1.62]* | *1.00* | — | — |

Two things worth reading carefully:

**A high Sharpe is not the same as alpha.** The long-tilted book's Sharpe of
1.06 comes with a beta of 0.91 — it is the market wearing a strategy's name.
The beta-neutral book has a lower Sharpe and far more alpha.

**The gradient-boosted model is the worst performer.** Its IC is −0.0009
(*t* = −0.11), which is zero, not backwards. It loses because it pays roughly
10%/yr in trading costs to trade noise. A six-line theory-driven composite with
no fitted parameters beats it comfortably.

### Portfolio contribution

Comparing a market-neutral book's raw return to equities is the wrong test —
nobody holds it *instead of* their equity allocation. Blended into a passive
portfolio at a 35–60% weight, it moves portfolio Sharpe from **1.06 to 1.15**.
That improvement's own confidence interval contains zero.

---

## The seven measurement biases

Each was found in this project's own output and verified against synthetic data
with known ground truth.

| # | Bias | Effect when uncorrected |
|---|---|---|
| 1 | Overlapping daily observations treated as monthly portfolio returns | Sharpe inflated **1.8×**, turnover understated |
| 2 | Z-scored features used as raw inputs to a ratio | Signal sign flipped on ~50% of rows |
| 3 | Cross-sectional mean of a z-score used as a regime input | Input had standard deviation 0.000000 |
| 4 | IC function mapping undefined correlations to 0.0 | Five of nine years of a baseline silently unmeasured |
| 5 | Sector taxonomy mismatch across two label sets | Each sector split in half; peer groups halved |
| 6 | Benchmark misaligned by one period against forward returns | Manufactured **+5.8%/yr of phantom alpha** |
| 7 | Dollar-neutral construction carrying −0.21 uncompensated beta | Cost ~3%/yr and masked a positive alpha |

Bias 6 is the instructive one. A long-only equity book was reporting a market
beta near **zero**, which is impossible. The benchmark had been built with
`pct_change()` on the same index as the strategy's *forward* returns, so the
two series were offset by one period and the regression compared
non-overlapping windows. On a synthetic book constructed with a known beta of
1.0 and zero true alpha, the faulty code reported beta −0.10 and **+15%/yr of
alpha**. The notebook now carries a beta invariant that fails loudly when a
book's measured exposure disagrees with its stated construction.

---

## Principal finding: alpha decay

Splitting the sample into fixed sub-periods, with the configuration chosen
before the split:

| Period | Alpha | Sharpe | Market |
|---|---:|---:|---:|
| 2010–2014 | **+2.8%** | 0.61 | +17.4% |
| 2015–2019 | +0.8% | 0.20 | +12.7% |
| 2020–2023 | **+0.2%** | 0.17 | +16.1% |

A 36-month rolling alpha falls from +3.4% to +0.3%, a trend of −0.36% per year
of elapsed time, and only 9% of rolling windows reach |*t*| > 2.

This is the **opposite** of what overfitting produces. Overfitting flatters the
period a model was tuned on — usually the most recent. Here the edge is
strongest in the oldest data and absent in the newest, which is the expected
fate of a published anomaly as it becomes widely traded. Short-term reversal
and the low-volatility effect have both been cheaply tradeable since roughly
2010, which is where the decline begins.

---

## Construction

The single largest improvement came from the portfolio, not the model.

| Change | Effect |
|---|---|
| Quintile buckets → rank-proportional weighting | **+0.45 Sharpe**, no change to the signal |
| Dollar-neutral → beta-neutral legs | Return −0.93% → +1.96%; drawdown −31.5% → −14.6% |
| Signal-rank → risk-parity sizing | Volatility −25% at unchanged return (≈ +0.22 Sharpe) |
| Vol-target leverage 1.83× → 1.0× | Cost 10.6%/yr → 4.3%/yr |

A quintile book holds only the top and bottom fifths and ignores 60% of the
ranking. In this data most of the signal sat in the middle.

---

## Ideas tested and rejected

| Idea | Outcome |
|---|---|
| Higher trading frequency | Break-even IC rises to 0.376 weekly against 0.012 achieved — 32× short |
| Higher-frequency training data | No gain. Overlapping samples add rows, not information |
| Drawdown de-risking overlay | Return autocorrelation −0.021 (*t* = −0.28): no clustering, so no Sharpe benefit |
| Fitted (IC-weighted) signal weights | IC −0.0185 against +0.0117 for equal weighting — overfits |
| Single-name risk caps | No effect at 5%, 2% or 1%; the book is already diffuse |
| Downside-volatility sizing | Correlates +0.92 with total volatility — nearly the same book |
| Sign-flipping the signal | Looked like a 2-point IC gain in-sample, reversed out of sample |

The fitted-weights result is worth a note. A synthetic test predicted
IC-weighting would *add* +0.0022. On real data it *subtracted* 0.030. The
diagnostic that explains why is weight stability: components with real
signal received stable weights across the whole sample, while the two whose
true IC was near zero flipped sign repeatedly. Fitted weighting was not
uniformly overfitting — it was fitting noise on the components that had no
signal to fit.

---

## Limitations

- **Not statistically significant.** The headline alpha *t*-statistic is 1.72,
  below the conventional bar of 2. Roughly 116 strategy variants were scored
  against the development sample, and the deflated Sharpe that accounts for
  that search is 0.32.
- **Survivorship.** 35% of point-in-time constituents (345 tickers, ~29,000
  constituent-months) had no price history available from the free data source.
  These are disproportionately delisted or acquired companies, and the bias
  falls on the short leg specifically.
- **Capacity.** The result depends on running at roughly $25–100M. At $500M,
  costs of ~10.7%/yr exceed any gross return this signal produces.
- **Fundamentals unusable.** The free data source reaches back about five
  quarters, so valuation and quality features were dropped on a coverage gate
  rather than imputed.
- **The holdout failed.** This is stated at the top rather than buried here.
  Development Sharpe +0.55 became −0.86 out of sample. Note what the
  composite's holdout does and does not test: with no fitted parameters there
  is nothing to overfit, so it tests whether the *effect persists*, not whether
  a *model generalises*. The machine-learning book's holdout is the test of
  generalisation, and it also failed — though it was already negative in
  development.

---


---

## Process

Research design, direction and review by the author; implementation and
analysis carried out with AI assistance. Every figure quoted in this README is
produced by the notebook at run time rather than typed by hand, and the
research log in Appendix A records each experiment and its outcome, including
the ones that failed.

## Running it

```bash
pip install yfinance xgboost hmmlearn statsmodels shap pandas numpy scikit-learn
```

Open the notebook and run all cells. Everything network-bound is cached to
`CFG.cache_dir`, so the first run takes 30–60 minutes (mostly data download)
and subsequent runs a few minutes. No API keys are required; an optional
Financial Modeling Prep key improves fundamental coverage.

Every parameter lives in one `Config` dataclass — cost model, universe, holdout
date, construction choices — so each assumption is visible in one place.

### Structure

| Part | Contents |
|---|---|
| **I — Data and Model** | Point-in-time universe, feature engineering, walk-forward training, signal quality |
| **II — Portfolio Construction** | Weighting, risk parity, beta neutrality, cost model, candidate strategies |
| **III — Results** | Capacity curve, sub-period stability, rolling alpha, portfolio contribution |
| **IV — Validation** | Significance testing, scorecard, out-of-sample holdout |
| **Appendix A** | Research log: every experiment and its outcome |

`Key Terms and Notes.md` explains every term used, from Sharpe ratio to deflated
Sharpe to coefficient rotation.

---

## What would move this forward

Not another model. The capacity sweep showed out-of-sample IC was
indistinguishable from zero at every model complexity from decision stumps to
depth-5 trees, so the constraint was never the learner. The remaining
constraints are data:

- **Fundamentals with real history**, rather than five quarters
- **A universe with larger anomalies** — small caps show stronger effects,
  though with worse costs; the capacity analysis here gives the framework to
  evaluate that trade directly
- **Intraday data**, where measurable effects are larger relative to costs

---

## Conclusion

Publicly available daily price data on large-cap US equities does not support a
market-neutral strategy distinguishable from zero after realistic costs over
this period. The signal that did exist decayed steadily from 2010 onward and
had **gone negative by the 2024–26 holdout**.

The value of the work is the measurement framework and the dated decay result,
not a deployable strategy. A pre-registered holdout that returns a negative
answer is a working experiment, not a failed one — the alternative is a
strategy that looks good until it is traded.

Several configurations had **positive IC and still lost money** —
because a quintile book discards the middle of the ranking, the short leg
carried uncompensated beta, and leverage multiplied a 10%/yr cost drag. Those
were portfolio problems rather than model problems, which mattered, because a
better model was never available from this data.
