---
name: benchmark-noise-floor
description: Use before trusting any backtest or performance metric, especially one measuring a small effect like execution cost, slippage or alpha per trade. Replays the metric under a counterfactual where the true answer is known to be zero, and measures what it reports anyway - the noise floor. Trigger on "is this result significant", "measure slippage/execution quality", designing a new performance metric, or when a metric's numbers swing wildly between periods.
---

# Measure what your metric reports when the answer is zero

## The test

Before believing a metric, **replay it over real data with the effect set to
exactly zero** - no cost, no slippage, perfect fills - and look at the spread of
what it returns. That spread is the metric's noise floor. If it is larger than
the effect you are hunting, the metric cannot see the effect, no matter how many
decimal places it prints.

## Why this is not optional

A real case: an experiment measured execution cost by comparing fills against the
official opening and closing auction prints - the convention in the literature.
Replaying 20 real sessions with **zero execution cost** still produced a standard
deviation of **41.6 bps** on that metric, against a target effect of about 1 bp.
The benchmark was ~98% noise. Every pre-registered threshold sat inside it.

The fix was to change the benchmark, not the sample size: measuring fills against
the consolidated quote midpoint **at the instant the order was sent** cut the
noise to ~4 bps. Same trades, same data, a metric that can actually resolve the
thing.

Live results since bore this out. Over eight sessions the two benchmarks ran:

| | execution vs arrival mid | auction vs prints |
|---|---|---|
| range | −0.7 to −3.0 bps | −45.6 to +94.1 bps |

The second column is the market moving overnight, not execution quality.

## How to run it

1. **Define the zero-effect counterfactual.** Usually: assume fills occur exactly
   at some reference price, so the true cost is zero by construction.
2. **Replay over real historical data**, not simulated - you want the actual
   volatility structure.
3. **Report the standard deviation, and the minimum detectable effect** at your
   planned sample size: roughly `(1.96 + 0.84) * sd / sqrt(n)` for 5%
   significance at 80% power.
4. **Compare that against the effect you expect.** If MDE > effect, stop and
   redesign; more sessions rarely closes a 40x gap.
5. **Keep both metrics** if the noisy one is the convention - report them side by
   side and difference them into a named term (e.g. `timing_drift_bps`) so nobody
   can conflate them later. Pin the identity with a test.

## Signals that you are measuring noise

- The metric's sign flips between adjacent periods while the underlying process
  is unchanged (−34 bps one night, +22 the next).
- Its magnitude is an order of magnitude larger than any plausible mechanism.
- It correlates with market movement rather than with the thing you changed.
- A confidence interval that spans zero comfortably at your full planned n - a
  reason to redesign before spending money, not after.

## Reporting rule

Quote the noise floor **next to** every result from that metric. "Execution cost
−2.2 bps (benchmark noise ~4 bps at n=8)" is honest; the number alone is not.
