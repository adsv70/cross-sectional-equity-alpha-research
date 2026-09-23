# Finance jargon in this project, explained

*Companion reference for the notebook in this repository. Every term used in the analysis, explained from first principles.*

Every term I used in the review or that appears in the notebook, in plain
language, with a pointer to where it shows up in the code. Read the first
section top to bottom; after that, dip in as needed.

---

## 1. What the strategy is actually doing

**Equity** just means stock. **Cross-sectional equity strategy** means: every
month, look at ~500 stocks, rank them from most attractive to least, buy the top
group and sell short the bottom group. One are not predicting whether the market
goes up. One are predicting which stocks beat which *other* stocks. That is the
"cross-section" — the slice across all stocks at one moment in time.

**Long** means one owns it and profit if it rises. **Short** means one borrowed
someone else's shares, sold them, and owe them back — so one profit if the price
falls. Shorting is how a strategy can make money in a falling market, and it is
why the notebook has a **short borrow cost** (see §4).

**Long/short book.** "Book" is trader-speak for portfolio. A long/short book
holds both. Yours targets **$1 long and $1 short**, which is why the code keeps
asserting each leg sums to 1.0.

**Net exposure** = long minus short. Yours is ~0, called **dollar-neutral** or
**market-neutral**: if the whole market drops 10%, the longs and shorts both
drop and roughly cancel. One have removed the market from the bet and kept only
the stock-picking. **Gross exposure** = long plus short = 2.0 in the case.
That distinction matters for costs: one is trading $2 of stock for every $1 of
capital.

**Leg.** One side of the book. "Long leg," "short leg."

**Cross-sectional vs. time-series.** The main strategy is cross-sectional: it
compares stocks to each other. Section 10 is **time-series** (also called
*absolute*) momentum: each stock's own history decides its own position, with no
reference to any other stock. That is why Section 10 is not dollar-neutral —
if most stocks are trending up, one end up net long, which is a market bet.
Worth knowing, because a market bet's returns are not comparable to a
market-neutral one's.

**Universe.** The set of stocks one is allowed to trade. Yours is S&P 500
members. **Constituent** = a member of an index. **Membership window** = the
dates a company was actually in the index.

**Rebalance.** Recomputing the positions. Yours is monthly, at month end.

**Holding period / horizon.** How long one holds before re-deciding. Yours is 21
trading days ≈ one month. (Markets are closed weekends and holidays, so a
"month" is ~21 trading days, not 30 calendar days. The notebook is careful
about this and it is right to be.)

**Forward return.** The return *after* the decision point — what one is trying
to predict. `fwd_ret` in the panel is the next 21 trading days' return.

---

## 2. Building the signal

**Signal / alpha.** A signal is any number one compute that one hope predicts
returns. **Alpha** is the return one earn that cannot be explained by just
taking obvious risks — the part that is genuinely yours. Pronounced like the
Greek letter; the term comes from the intercept in a regression (see §6).

**Factor.** A characteristic shared by many stocks that explains returns:
size, value, momentum, quality, low volatility. Factors are the well-known,
widely-harvested effects. Alpha is what is left *after* removing them. A
strategy that looks profitable but is really just a momentum factor bet has no
alpha — one could buy that exposure in an ETF for a few basis points.

**Momentum.** The tendency of stocks that went up to keep going up, over
horizons of roughly 3–12 months. `mom_63`, `mom_252` in the code (63 trading
days ≈ 3 months, 252 ≈ 1 year).

**Short-term reversal.** The *opposite* effect at short horizons: stocks that
jumped hard over the last week or few weeks tend to give some back. Usually
explained as compensation for absorbing a big one-sided order. This is the
effect I flagged as missing from the feature set, and a likely explanation for
the persistent negative IC — the model may be trading momentum at a horizon
where reversal dominates.

**12-1 / skip-month momentum.** `mom_252_skip1` measures the last 12 months of
return but *excludes* the most recent month, precisely to avoid contaminating
momentum with short-term reversal. The notebook already does this correctly.

**Volatility.** How much a stock's price bounces around, usually the standard
deviation of daily returns, **annualized** (multiplied by √252 to express it as
a yearly figure). `vol_21` is the last 21 days' volatility, annualized. Higher
volatility = wider range of outcomes = more risk.

**Risk-adjusted momentum.** Return divided by volatility. The logic: a 5% gain
in a calm stock is more meaningful than a 5% gain in a wild one. This is what
Section 10 *intended* to compute and did not, because both inputs had already
been z-scored.

**RSI (Relative Strength Index).** A bounded 0–100 oscillator built from the
ratio of recent gains to recent losses. Above 70 conventionally means
"overbought," below 30 "oversold." It is a technical-analysis staple; treat it
as just another momentum-ish feature.

**Fundamentals.** Numbers from a company's financial statements rather than its
price:
- **P/E ratio** (price to earnings): share price ÷ annual earnings per share.
  Low can mean cheap, or can mean the business is deteriorating.
- **P/B ratio** (price to book): price ÷ accounting net worth. Same caveat.
- **ROE** (return on equity): profit ÷ shareholder equity. A profitability measure.
- **Profit margin**: profit ÷ revenue.
- **Debt-to-equity**: leverage. How much borrowed money is in the capital structure.

**Value** as a factor means buying low P/E, low P/B stocks. **Quality** means
buying high-ROE, high-margin, low-debt stocks.

**Z-score.** `(x − mean) / standard deviation`. Converts any number to "how many
standard deviations from average." The notebook z-scores within each
sector-date group so that a tech stock's momentum is compared to other tech
stocks on the same day, not to a utility six months earlier. This is standard and
correct — the bug was that later code kept using the columns as if they were
still raw prices and returns.

**Winsorization.** Clipping extreme values to a percentile (the code clips at
the 1st and 99th). Stops one crazy outlier from dominating. Named after a
statistician, Charles Winsor. Note the code winsorizes *features* but not the
*target*, which is the fat-tails problem I flagged.

**Sector-neutral.** Comparing and trading stocks only against others in the same
sector, so one is not accidentally making a bet like "tech beats energy." Done
two ways in the code: z-scoring within sector, and picking the top/bottom names
within each sector.

**Interaction feature.** A feature built by multiplying two others
(`mom_vol_interact = mom_252 × vol_21`), to let a model express "momentum
matters more when volatility is low."

---

## 3. Building the portfolio

**Quantile / quintile.** Sort stocks into equal-sized buckets. Quintiles = 5
buckets. `CFG.n_quantiles = 5`. One buy Q4 (top) and short Q0 (bottom).

**Monotonicity.** A good signal produces a **monotone ladder** across quantiles:
Q0 lowest average return, then Q1, Q2, Q3, Q4 highest, steadily increasing.
That is evidence the ranking carries directional information. The ladder is
**U-shaped** (both ends high, middle flat), which is the signature of a model
sorting on volatility rather than direction. See T4 in the test pack.

**Equal weight vs. signal weight.** Equal weight gives every selected name the
same dollar amount. Signal weight (`use_signal_weighted_sizing`) gives more to
higher-conviction names.

**Cap-weighted.** Weighted by company size. The S&P 500 itself is cap-weighted,
which is why SPY is the fairer benchmark than an equal-weight universe average.

**Benchmark.** What one compare against. If the strategy returns 8% and the
index returned 12%, one lost, however good 8% sounds.

**Volatility targeting.** Scaling the whole book up or down to hit a chosen risk
level. The config targets 10% annualized volatility: if the strategy has been
calm lately, lever up; if wild, cut back. **Leverage** here means trading more
than $1 of stock per $1 of capital, capped at 2× in the config.

**Turnover.** How much of the portfolio one replace each rebalance. **One-way
turnover** of 100% means one replaced essentially every position. Yours is ~102%
per month, which is very high and is why trading costs dominate the results.

**Capacity.** How much money a strategy can run before its own trades move
prices against it. `assumed_aum_usd = $500M` exists only to convert portfolio
weights into dollar trade sizes so capacity can be checked.

**ADV (average daily volume).** How much of a stock trades in a typical day.
If one need to buy $10M of a stock that trades $50M a day, one is 20% of the
volume and will push the price. The `max_position_pct_adv = 0.05` caps each
position at 5% of that name's daily volume.

**Participation rate.** The trade size as a percentage of ADV. The core input
to market-impact cost.

---

## 4. Costs and frictions

**Basis point (bp).** One hundredth of a percent. 10 bps = 0.10%. Universal
finance shorthand; `transaction_cost_bps = 10` means 0.10% per trade.

**Transaction costs.** Everything that makes a trade cost more than the quoted
price: commissions, the bid-ask spread, and market impact.

**Bid-ask spread.** At any moment there is a price buyers will pay (the bid) and
a higher price sellers will accept (the ask). One buy at the ask and sell at the
bid, so one loses the difference on every round trip.

**Market impact.** The own buying pushes the price up before one finish. Bigger
trades relative to ADV cost more. The v12 switched from a linear to a
**square-root** impact model, which is the practitioner standard — the second
$1M of a trade costs less extra than the first did.

**Slippage.** The general term for executing at a worse price than one expected.

**Short borrow cost.** To short a stock one must borrow the shares and pay a fee
for as long as one holds the position. Yours assumes 50 bps a year on gross short
exposure. Critically this is a **holding** cost, not a trading cost — one pays it
every month whether or not one traded. The notebook models this correctly,
which many don't.

**Cost drag.** How much costs subtract from returns. Yours is ~0.42% a month ≈
5% a year, which is more than most equity strategies earn in total. This is why
the turnover number matters so much.

---

## 5. Measuring performance

**Return.** Percentage gain. **Annualized return / CAGR** (compound annual growth
rate): the constant yearly rate that would produce the same final result.

**Volatility (of a strategy).** Standard deviation of returns, annualized. The
standard risk measure.

**Sharpe ratio.** Return ÷ volatility, annualized. How much return per unit of
risk. The single most-quoted number in the industry. Rough scale: below 0.5 is
weak, 1.0 is good, above 2.0 sustained is rare enough to be suspicious. It is
also the number most easily inflated by a measurement bug, which is what was
happening in the Sections 10 and 12.

**Drawdown.** How far one is below the previous peak. **Maximum drawdown** is
the worst such fall. A 50% drawdown is psychologically and often
professionally fatal even if the strategy eventually recovers.

**Calmar ratio.** Annual return ÷ maximum drawdown. A Sharpe-like measure that
punishes deep holes rather than general bumpiness.

**Hit rate.** Fraction of periods with a positive return. Useful but shallow —
one can win 70% of months and still lose money if the 30% are big.

**Information Coefficient (IC).** The workhorse metric in this notebook. Each
month, compute the **Spearman rank correlation** between the predictions and
what actually happened. Spearman means it only cares about *order*, not
magnitude — did one rank the winners above the losers? Ranges −1 to +1. In
cross-sectional equities, **0.02 to 0.05 is a genuinely useful signal** and 0.10
would be extraordinary. Yours is −0.0217, meaning the model ranks slightly
backwards, consistently.

**IC Information Ratio (IC IR).** Mean IC ÷ standard deviation of IC,
annualized. Measures *consistency* of the signal, not just its average
strength. A signal with a small but stable IC can be more valuable than a
larger, erratic one.

**Information Ratio (IR).** Confusingly, also used for a strategy's excess
return over a benchmark ÷ the volatility of that excess. Context tells one
which meaning is in play.

---

## 6. Statistics needed for this

**t-statistic.** How many standard errors an estimate sits from zero. |t| > 2 is
the conventional "probably not luck" threshold (it corresponds to roughly a 5%
chance of seeing something that large if the truth were zero).

**p-value.** The probability of seeing a result this extreme if there were
genuinely no effect. Below 0.05 is the conventional bar. Both t and p answer the
same question in different units.

**Newey-West.** A correction to the t-statistic for when the observations are
not fully independent of each other — which is true of consecutive months of a
strategy's returns. Without it, t-stats come out too big and one get
overconfident.

**Regression.** Fitting `y = a + b₁x₁ + b₂x₂ + …`. In **factor attribution**,
y is the strategy's monthly return and the x's are factor returns. The **betas**
(b) say how much of each factor one is carrying. The **intercept** (a) is the
**alpha** — the part not explained by any of them. **R²** says what fraction of
the returns the factors explain; a high R² with near-zero alpha means the
"strategy" is a repackaged factor bet.

**Beta.** Sensitivity to something, usually the market. Beta of 1.0 moves with
the market one for one; 0 is market-neutral.

**Skew.** Asymmetry of a distribution. Stock returns are **right-skewed**: most
stocks do mediocrely, a few do spectacularly. This matters in the notebook
because it makes the cross-sectional **mean** higher than the **median**, so a
target demeaned by the median has a positive average — which the short leg pays
for every month.

**Kurtosis / fat tails.** How often extreme outcomes happen relative to a normal
bell curve. Financial returns have very fat tails. This is why training a
squared-error model on raw returns is a bad idea: it spends its effort fitting
the ±50% months, which are mostly noise.

**Rank / percentile transform.** Replacing values with their rank position.
Destroys magnitudes, keeps order, immune to outliers. Since IC only measures
order anyway, training on ranks costs one nothing and buys robustness.

---

## 7. Ways a backtest misleads

A **backtest** is simulating a strategy on historical data. Nearly every way it
can mislead one has a name.

**Look-ahead bias.** Using information one would not have had at the time. The
classic version: a company reports Q1 earnings in mid-May, but the data file
stamps them "March 31," so the backtest trades on them six weeks early. The
notebook handles this with `REPORTING_LAG_DAYS = 45` and `available_date`,
which is correct practice.

**Survivorship bias.** Only including companies that still exist. If one build a
universe from today's S&P 500 and run it back to 2014, every company that went
bankrupt is missing and the returns look great. The notebook mostly avoids
this with a **point-in-time (PIT)** universe — reconstructing who was actually
a member on each date — which is genuinely good work. But 180 of 784 tickers
were dropped because Yahoo has no data for them, and those are
disproportionately the failures. So a residual version of the bias remains.

**Point-in-time (PIT).** Data as it was known on the date, not as later revised.
Economic statistics and financial statements both get restated; a PIT dataset
remembers the original.

**Data snooping / overfitting.** Trying many variants on the same data until one
looks good, then believing it. The more one try, the more certain it is that
the best result is luck. The Section 11 caught a textbook case: a sign-flip
looked like a 2-percentage-point IC improvement in-sample and reversed out of
sample.

**In-sample vs. out-of-sample.** In-sample is data the model saw while being
fit. Out-of-sample is data it did not. Only out-of-sample results mean anything.

**Walk-forward.** Repeatedly: train on everything up to month T, predict month
T+1, roll forward. This is the correct way to simulate a strategy one would
actually have run, because at no point does the model see its own future.
**Expanding window** means the training set grows forever; **rolling window**
means it keeps a fixed recent length.

**Embargo.** A gap between the end of the training data and the start of the
test data. Necessary here because the target is a 21-day *forward* return — a
training row dated the day before the test date already contains information
about the following three weeks. The notebook implements this correctly.

**Overlapping returns.** When consecutive observations share most of their
measurement window. Daily rows with a 21-day forward return overlap by 20 of 21
days. They look like independent data points but carry almost the same
information, which makes models overconfident and Sharpe ratios inflated. This
was the second bug in the Sections 10 and 12.

**Deflated Sharpe ratio.** Adjusts a Sharpe for how many variants were searched.
The calibration I ran: on 104 months, a realised Sharpe of +0.83 scores 96.9%
confidence if it was the only idea, and 2.8% if it was the best of thirty.

**Pre-registration.** Writing down which experiments one will run *before*
running them, so one cannot quietly redefine success afterwards. Borrowed from
clinical trials. The T3 grid in the test pack is pre-registered: four configs,
named in advance.

**Holdout.** A slice of data one lock away and never look at until the very end.
The single most effective defence against everything in this section.

---

## 8. Filtering and regime-detection techniques

These are statistics/engineering methods rather than finance, but one asked.

**Kalman filter.** A recursive way to estimate a hidden quantity — here, a
stock's underlying trend — from noisy observations. At each step it predicts,
then corrects using the new observation, weighting the correction by how much
it trusts each. Crucially it is a **filter**, not a **smoother**: the estimate
at time t uses only data up to t, so it cannot look ahead. The implementation
is correct and genuinely leak-free.

**Hidden Markov Model (HMM).** Assumes the world is in one of a few unobserved
"states" (here: trending vs. choppy), each producing different return behaviour,
with some probability of switching. **Viterbi** is the algorithm that finds the
most likely sequence of states given what one observed. **EM fitting** is how
the parameters are estimated, and it is sensitive to its starting point —
which is exactly the problem the notebook found and fixed with multiple random
restarts. Good catch, kept in the patch.

**Bayesian change-point detection.** Asking, statistically, "did the process
generating this data just change?" rather than reacting to every wiggle. The
implementation compares two hypotheses — recent observations came from the same
distribution as before, versus a different one — and only flips a position when
the posterior probability of a real change crosses a threshold. The maths is
correct.

**Posterior probability.** In Bayesian statistics, the belief in a hypothesis
*after* seeing the evidence, as opposed to the **prior** (before). A 50/50 prior
means one started neutral.

**Conjugate prior / Normal-Inverse-Gamma.** A mathematical convenience: choosing
a prior whose form survives the update, so the answer has a closed formula
instead of needing simulation. That is what `_log_marginal_likelihood_normal`
implements.

**Avellaneda-Stoikov.** A market-making model: where should a dealer post its
buy and sell quotes given that it is holding inventory it would rather not?
**Market maker** = someone who continuously quotes both a buy and a sell price
and earns the spread. **Order book** = the live list of all resting buy and sell
orders. **Inventory risk** = the danger of being stuck holding a position one
did not choose. The notebook's judgment call here was right: without order-book
data and with monthly rebalancing, a literal implementation is not possible, and
transplanting only the inventory-aversion concept — and labelling it as such —
is the honest move.

**SHAP values.** A method for attributing a model's prediction to its individual
inputs. Answers "why did the model say this?" rather than "which features does
it use on average."

---

## 9. Shorthand used in the analysis

**"The book"** = the portfolio.

**"Leg"** = one side of it.

**"Dollar-neutral"** = equal long and short.

**"Sizing"** = deciding how big each position is.

**"Conviction"** = how strongly the model believes, used to size positions.

**"Drag"** = anything steadily subtracting from returns.

**"Ablation"** = removing components one at a time to see what each contributes.
Borrowed from machine learning.

**"Uplift"** = incremental improvement.

**"Ladder"** = the table of average returns by prediction quantile.

**"Trials budget"** = how many strategy variants one can test before the
results stop meaning anything.

**"Artifact"** = a number produced by how one measured rather than by what one
measured. Both of the Section 10 problems produce artifacts.

**"Signature"** = a recognisable pattern that points to a specific cause. A
U-shaped ladder is the signature of a volatility sort.

---

## 10. Feature families and model diagnostics

### The new feature families

**MAX effect / lottery demand.** Stocks that posted one unusually large daily
gain in the recent past tend to underperform. The explanation is that some
investors treat stocks like lottery tickets and overpay for the chance of a big
jump. `max_ret_21` is the negative of the largest daily return in the past
month, so a **positive** value means "no recent spike."

**52-week-high proximity.** How close a stock trades to its highest price of
the past year. Stocks near their 52-week high tend to keep outperforming. This
is an **anchoring** effect — traders treat the old high as a reference point and
are slow to bid past it — and it is distinct from momentum, which is about the
rate of change rather than the level.

**Idiosyncratic volatility (`ivol`).** Total volatility minus the part
explained by the market. One first estimate the stock's **beta**, subtract
`beta × market return` from each day's return, and take the standard deviation
of what is left. That residual is the risk specific to the company. The
low-risk anomaly is usually stated in terms of idiosyncratic rather than total
volatility.

**Amihud illiquidity.** Average of `|daily return| ÷ dollar volume`. It measures
how far the price moves per dollar traded, so a high value means a thin,
easily-pushed stock. Illiquid stocks tend to earn a premium as compensation for
being hard to exit.

**Return skew.** The asymmetry of a stock's own daily return distribution.
Negatively-skewed stocks — occasional large drops — tend to earn a premium for
carrying that crash risk.

### Diagnosing the model

**Model capacity.** How much structure a model is able to express. For
gradient-boosted trees it is mainly tree depth (how many features can interact
in a single rule), the number of trees, and the minimum observations a leaf
must contain. High capacity lets a model fit real structure — and also lets it
fit noise.

**Decision stump.** A tree of depth 1: one split on one feature. A boosted
ensemble of stumps is a weighted sum of single-feature rules with **no
interactions at all**. If out-of-sample performance peaks there, it is evidence
that there are no interactions worth finding in the data.

**Memorisation.** Fitting patterns specific to the training sample that do not
recur. The signature is a large **in-sample / out-of-sample gap** — which is
what D1 found: in-sample IC +0.139 against out-of-sample −0.019.

**`min_child_weight` / minimum leaf size.** How many observations a tree leaf
must contain before the model is allowed to make a rule from it. Raising it
forbids the model from carving out tiny, hyper-specific groups, which is one of
the most direct ways to cut memorisation.

**Column subsampling (`colsample`).** The fraction of features each tree is
allowed to look at. Lower values force variety across trees. Note the
interaction with dead features: if a third of the columns carry no
information, a 0.5 colsample means a third of what each tree sees is noise.

### Costs and viability

**Break-even IC.** The signal quality a strategy needs simply to cover its own
trading costs, before earning anything. Computed as
`annual cost drag ÷ return per unit of IC`. This is the single most
decision-relevant number in the project: at monthly turnover the bar was an IC
of **0.047**, at semiannual holding **0.015**. Nothing here has ever produced
an IC above 0.02, which is why turnover reduction matters more than modelling.

**Return per unit of IC.** How much gross return one unit of ranking skill
actually buys. It depends on the universe's return dispersion and on how the
portfolio is built, so it is estimated from the own data rather than assumed.
In the v14 sample each 0.01 of IC was worth about 1.5% a year.

**Cost drag.** Annualised trading costs as a percentage of capital. 7.44%/yr in
the v13 run — more than most equity strategies earn in total, and 59% of the
total loss.

**No-trade band / hysteresis.** A buffer that stops a portfolio churning on
names hovering at a selection boundary. A stock must cross a **narrow entry**
threshold to get in, but is kept while it stays inside a **wider exit**
threshold. Widening the gap between them reduces turnover at the cost of
holding slightly staler positions.

**Holding period.** How long positions are kept before the book is re-decided.
Distinct from the **forecast horizon**, which is the period the signal predicts
over. v14 predicted 21 days ahead and rebalanced every 21 days; T14 tests
separating them.

### Research-process terms

**Adaptive sign.** Deciding whether to trade a signal long or inverted using
only information available at the time — here, the trailing realised IC. This
is the honest counterpart to looking at a full-sample result and flipping it,
which is hindsight rather than a strategy.

**Ablation.** Removing or adding components one at a time to see what each
contributes. Borrowed from machine learning. It is how T6 established that the
change-point gate did all the work in Section 12 and the other two components
did none.

**Bonferroni correction.** When one test many hypotheses at once, some will
clear the usual significance bar by luck alone. Bonferroni raises the bar in
proportion to the number of tests — with 114 comparisons, the |t| threshold
rises from 2.0 to about 3.5.

**Pre-registration.** Writing down which experiments one will run before
running them, so success cannot be redefined afterwards. Borrowed from clinical
trials.

**Trials budget.** How many strategy variants one can test against one sample
before the best result is just the maximum of that many noisy draws. It feeds
directly into the deflated Sharpe ratio.

**Contamination.** When a data or code fault makes a run's results
unreliable — as when v14's sector coverage collapsed to 33.4% and every
sector-neutral operation silently stopped working. Contaminated results are not
"a bit noisy"; they are not comparable to anything.

**Taxonomy mismatch.** Two data sources using different labels for the same
category — `"Financials"` versus `"Financial Services"`. Left unnormalised, the
same sector gets split in two and every sector-relative comparison is made
against half the real peer group. This bug sat in the notebook from v12 to v14.

---

## 11. Portfolio construction

**Quintile portfolio.** Sort into five buckets, buy the top, short the bottom,
ignore the middle three. Standard, but it discards 60% of the ranking — and in
this project most of the signal sat in the middle.

**Proportional (rank) weighting.** Weight every name by its centred
within-sector rank instead of using two equal-weight buckets. This is what
Grinold's fundamental law assumes when it says `IR ≈ IC × √breadth`; **breadth**
means every position one holds, not just the extremes. Worth +0.45 Sharpe here.

**Risk-parity sizing.** Divide each weight by the name's volatility so every
position contributes the same amount of *risk*. Without it, volatile names
dominate the variance while carrying the same expected return.

**Beta-neutral versus dollar-neutral.** One dollar long against one dollar
short is not zero market exposure when the legs hold different kinds of stock.
Beta-neutral sizing scales the legs so `Σ(w_long × β) = Σ(w_short × β)`. In
this project a supposedly market-neutral book carried a **−0.21** beta, which
in a +14%/yr market cost about three points a year and hid a positive alpha.

**Gross-preserving neutralisation.** Scaling *both* legs so beta nets to zero
while total gross exposure stays fixed. Scaling only one leg silently raises
gross — and therefore cost and risk — without that appearing anywhere.

**Trimming and smoothing.** Rank weighting gives every name a position, so
every name trades every month. Trimming drops the middle, where weights are
negligible; smoothing moves only part-way to target each period. Both cut
turnover; only trimming discards information.

**One-sided volatility targeting.** Scale exposure *down* when volatility
rises but never *up*. Classic vol targeting does both, which on a weak book
multiplies cost faster than exposure, since impact scales as dollars^1.5.

**Structural drag / regression intercept.** Regress a book's return on its own
IC; the intercept is what it earns at *zero* skill. Break-even IC must clear
this as well as costs.

---

## 12. Cost, capacity and viability

**Break-even IC.** The signal quality needed just to cover trading costs:
`(annual cost − intercept) ÷ return-per-unit-IC`. The single most
decision-relevant number in this project — roughly **0.094 at monthly
turnover**, against an achieved IC near 0.012.

**Return per unit of IC.** How much gross return one unit of ranking skill
buys, given the universe's dispersion and the portfolio's construction.
Estimated from the data rather than assumed.

**Cost elasticity to size.** Impact scales as dollars^1.5, so net Sharpe is
`(gross − cost·√k) / vol` for a book scaled by *k*. Net Sharpe **rises** as
size falls. This is why any rule that reduces average exposure looks like skill
in a cost-dominated book, and why capacity is a first-order property rather
than a footnote.

**Capacity.** The fund size above which a strategy's own trading destroys its
edge. The honest form of a result is "works below $X, at Sharpe Y" — not
"works" or "doesn't".

---

## 13. Diagnosing a model

**Model capacity.** How much structure a model can express: tree depth, number
of trees, minimum leaf size. High capacity fits real structure *and* noise.

**Decision stump.** A depth-1 tree — one split on one feature. A boosted
ensemble of stumps has **no interactions at all**. If out-of-sample
performance peaks there, no interactions exist worth finding.

**Memorisation.** Fitting patterns that do not recur. The signature is a large
in-sample / out-of-sample gap.

**Coefficient rotation.** Whether the feature-to-return relationship holds its
*direction* over time, measured by cosine similarity between fitted
coefficient vectors at different lags. A short **half-life** means models go
stale quickly. Distinct from decay in *strength*: a relationship can weaken
without turning, in which case more history still helps.

**Weight stability.** How much a fitted weight moves over time, and how often
it changes sign. A weight that flips repeatedly is tracking noise, not signal
— this is how to tell overfitting from genuine estimation.

**Alpha decay.** A real effect being arbitraged away. Distinguished from
overfitting by *when* it appears: overfitting flatters the period one tuned on
(usually recent), decay shows the edge strongest in the **oldest** data.

---

## 14. Research process

**Pre-registration.** Naming the experiments before running them, so success
cannot be redefined afterwards. Borrowed from clinical trials.

**Trials budget.** How many variants one can test against one sample before
the best result is just the maximum of that many noisy draws. Feeds directly
into the deflated Sharpe ratio.

**Bonferroni correction.** With many simultaneous tests some clear the usual
bar by luck; the threshold rises in proportion. With 114 comparisons, |t| goes
from 2.0 to about 3.5.

**Block bootstrap.** Resampling in contiguous blocks rather than individual
months, so the autocorrelation a strategy's returns actually have is
preserved. Month-by-month resampling pretends independence and produces
intervals that are far too narrow.

**Ablation.** Adding or removing components one at a time to see what each
contributes.

**Contamination.** When a data or code fault makes a run unreliable — not "a
bit noisy", but not comparable to anything.

**Look-ahead bias.** Using information unavailable at the time. The subtle
version here was a benchmark misaligned by one period against forward returns,
which manufactured 5.84%/yr of phantom alpha.

**Period mismatch.** Comparing strategies measured over different windows. A
strategy starting in 2009 against a benchmark starting in 2005 is comparing
two different decades — the benchmark absorbs 2008 and the strategy does not.

---

## 15. The numbers worth remembering

| Quantity | Value | Why it matters |
|---|---|---|
| Useful IC in cross-sectional equities | 0.02–0.05 | 0.10 would be extraordinary |
| Best IC achieved here | +0.018 (5-day reversal alone) | below the corrected significance bar |
| Break-even IC at monthly turnover | ~0.094 | roughly 8× what the signal delivers |
| Cost drag at $500M / $25M | 10.7% / 1.3% per year | capacity is the binding constraint |
| Construction gain, quintile → proportional | **+0.45 Sharpe** | no change to the signal |
| Beta-neutralisation gain | −0.93% → +1.96% return | removing uncompensated risk |
| Deflated Sharpe, 104 months, 1 vs 30 trials | 96.9% vs 2.8% at Sharpe +0.83 | the cost of searching |
| SPY over the same window | 16.6%/yr, Sharpe 1.06 | the free alternative, and the real bar |

### One paragraph

Publicly available daily price data on large-cap US equities does not support
a market-neutral strategy distinguishable from zero after realistic costs.
Several configurations had **genuinely positive IC and still lost money**,
because a quintile book discards the middle of the ranking, the short leg
carried uncompensated beta, and leverage multiplied a 10%/yr cost drag. Those
were portfolio problems, not model problems — which mattered, because the
capacity sweep showed a better model was never available from this data. The
signal that did exist decayed steadily from 2010 onward.

---

## 16. Defending this work in an interview

The concepts below are the ones most likely to be probed. Each entry gives the
question and **what a correct answer has to contain** — not a script. Scripted
answers fail on the first follow-up; derived ones do not. Work each out from
first principles until it can be reconstructed on a whiteboard without notes.

### Near-certain questions

**"Why does the embargo matter?"**
The target is a 21-*session forward* return. A training row dated one day
before the test date has a label that overlaps the test period almost
entirely, so the model would be fitted on the answer. The embargo cuts
training data off 21 sessions before each test date. A good answer states the
overlap explicitly rather than saying "to prevent leakage".

**"Why did overlapping observations inflate Sharpe by 1.8×?"**
Daily rows with a 21-day forward return share 20 of their 21 days. Averaging
21 overlapping windows produces a smoother series than 21 independent ones, so
measured volatility falls while mean return does not, and Sharpe rises. The
theoretical floor for that ratio is 0.817; the observed effect was larger
because the signal also changed within the month. Be able to derive why
averaging overlapping windows reduces variance.

**"What does the deflated Sharpe ratio adjust for, and why did you need it?"**
Searching many strategy variants against one sample means the best result is
partly the maximum of many noisy draws. The deflated Sharpe estimates the
probability the true Sharpe exceeds zero given the number of trials. Concrete
anchor: on 104 months, a realised Sharpe of +0.83 scores 96.9% at one trial
and 2.8% at thirty. Here, 116 variants were searched.

**"Your holdout failed. Why are you showing me this?"**
Because it was pre-registered, run once, and reported. The composite has no
fitted parameters, so this is not overfitting — it is the effect ceasing to
work, which the decay curve (+2.8% → +0.8% → +0.2% → negative) already
predicted. The alternative would have been a strategy that looked good until
it was traded.

### Likely follow-ups

**"Square-root impact — why not linear?"**
Impact scales with the square root of participation (trade size relative to a
name's own daily volume). Linear impact overweights high-participation trades.
The consequence used elsewhere in the project: total cost scales as
dollars^1.5, so net Sharpe is `(gross − cost·√k)/vol` for a book scaled by *k*
— which is why smaller books are genuinely cheaper, not just smaller.

**"Dollar-neutral versus beta-neutral?"**
One dollar long against one dollar short is not zero market exposure when the
legs hold different kinds of stock. The book here carried a **−0.21** beta
because its low-volatility tilt put low-beta names long and high-beta names
short. In a +14%/yr market that cost about three points a year and hid a
positive alpha. The fix scales both legs so their betas offset *and* total
gross exposure is unchanged.

**"Why is your Sharpe only 0.55 when the index did 1.06?"**
Different objects. The index is one large bet that the market rises; this book
has beta 0.03 and is not making that bet. The right comparison is what it does
to a portfolio: blended at 35–60% it moved portfolio Sharpe from 1.06 to 1.15,
though that improvement's own confidence interval contains zero.

**"What is IC and what counts as good?"**
Monthly Spearman rank correlation between prediction and realised return.
0.02–0.05 is genuinely tradeable in cross-sectional equities; 0.10 would be
extraordinary. This project's best was +0.018 on a single feature. Be ready
for the follow-up: IC alone is not enough — the break-even IC at monthly
turnover here was about 0.094, roughly eight times what the signal delivered.

**"How do you know the model wasn't just overfitting?"**
Two diagnostics. In-sample versus out-of-sample IC quantifies the gap
directly. The capacity sweep varied model complexity from decision stumps to
depth-5 trees and found out-of-sample IC indistinguishable from zero at every
setting — with the *deepest* model generalising best, which rules out capacity
as the constraint.

### The one to prepare hardest

**"Walk me through one of the seven bugs."**
Use the benchmark misalignment. The strategy's return at month *M* is measured
*forward* from month-end *M*; the benchmark was built with `pct_change()` on
the same index, so it covered month *M* itself. The two were offset by one
period and the regression compared non-overlapping windows, which collapsed
beta toward zero and turned the book's market return into apparent alpha. The
tell was a long-only equity book reporting a beta near zero, which is
impossible. Verified on synthetic data with a known beta of 1.0 and zero true
alpha: the faulty code reported beta −0.10 and **+15%/yr of alpha**.

That answer demonstrates the whole skill set — noticing an impossible number,
forming a hypothesis, and testing it against ground truth.

### Questions worth asking back

- How do they handle capacity constraints on strategies that work at small size?
- What do they consider an acceptable trials budget before a result stops being credible?
- How often do their signals decay, and how is that monitored?
