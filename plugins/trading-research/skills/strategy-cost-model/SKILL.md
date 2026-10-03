---
name: strategy-cost-model
description: Use when judging whether a trading strategy survives its real costs, comparing brokers, or choosing a stake size. Computes the break-even cost per round trip against the measured edge, then cost curves by broker and stake so the answer is a number rather than an opinion. Trigger on "is this strategy profitable after fees", "which broker should I use", "what stake makes this work", "the backtest shows X% a year", or any claim that applies an assumed cost per trade.
---

# Charge the strategy at measured costs, not assumed ones

## The failure this prevents

Published strategies - blog posts, Substack backtests, papers - almost always
show a large frictionless return and then apply an **assumed** cost per round
trip. That assumption decides the answer and is rarely measured. One real
example: an overnight-hold strategy showing 727% CAGR before fees and "around 2x
a year" after assuming 5 bps per leg. The real cost on the broker the reader
would actually use was 15 bps per leg, and the strategy was underwater.

**Your job is to replace the assumption with a measurement, then see if the edge
survives.**

## Method

1. **Measure the edge per round trip**, geometrically, over the real price
   series: `gross = prod(1 + r_i)` across n opportunities, edge per session is
   `gross**(1/n) - 1`.
2. **Compute the break-even cost** - the cost at which the whole period nets
   zero: `breakeven = 1 - exp(ln(1/gross) / n)`. Quote it in bps. This single
   number is the headline: any broker charging more is disqualified.
3. **Build cost curves by stake**, because fees have different shapes:
   - **Percentage fees** (FX conversion, stamp duty, spread) never improve with
     size. If they exceed break-even, no stake saves the strategy.
   - **Per-order minimums** (most commission schedules) are brutal when small and
     negligible when large. They set a *minimum viable stake*.
   - **Regulatory fees** are usually noise, but include them to be honest.
4. **Show the comparison as a table of bps by stake**, marking which cells clear
   the edge. That table is the deliverable; it makes the decision obvious.

## Worked example that shaped this skill

NVDA overnight hold, 126 sessions, edge 22.1 bps per session, break-even
**22.1 bps** per round trip:

| Stake | Percentage-fee broker (0.15%/leg) | Per-order minimum ($1/order) | Commission-free |
|---|---|---|---|
| $100 | 30 bps ✗ | 200 bps ✗ | 0.3 bps ✓ |
| $1,000 | 30 bps ✗ | 20 bps ✓ | 0.3 bps ✓ |
| $10,000 | 30 bps ✗ | 2 bps ✓ | 0.3 bps ✓ |

The percentage-fee broker is dead at every size; the minimum-fee broker needs
~$1,000+. That distinction is invisible until you draw the curve.

## Rules that keep the answer honest

- **Quote the break-even next to the edge.** "Costs 30 bps" means nothing until
  the reader sees the edge is 22.
- **Separate one-off from per-trade costs** and amortise the one-offs over the
  planned number of trades - a one-time currency conversion at 15 bps across 20
  sessions is 0.75 bps per session, not 15.
- **State the statistical strength of the edge.** Report the t-statistic
  (`mean / (sd / sqrt(n))`). Below about 2 the edge is indistinguishable from
  luck, and a break-even computed from it is a fiction. Say so.
- **Beware selecting the instrument after seeing its volatility** - that inflates
  every backtest. Note it when it happens.
- **Include the costs that do not look like costs**: currency conversion, stamp
  duty (0.5% on UK share purchases; AIM and ETFs exempt), and the spread itself.
  In the example above, conversion alone was 11x the execution cost.
- **Keep the cost constants in one place with their provenance in a comment**, so
  they are visibly measurements rather than guesses, and easy to re-run as real
  data arrives.
- **Use split-adjusted prices for any return series spanning a corporate action,
  and raw prices only for comparing against real fills.** These are different
  jobs and need opposite settings. A 10-for-1 split in a raw series reads as a
  -90% overnight return: in one real case it turned a genuine +567% five-year
  overnight series into -32.9%, which would have killed a strategy that worked.
  The tell is a single gap an order of magnitude larger than any other - sort the
  returns by absolute size and look at the top three before trusting a multi-year
  number.
