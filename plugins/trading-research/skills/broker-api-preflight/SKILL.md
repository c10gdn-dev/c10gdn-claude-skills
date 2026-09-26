---
name: broker-api-preflight
description: Use before automating any broker API, or when a trading integration behaves oddly - missing fills, unexpected fees, rate limits, currency conversions you did not ask for. A checklist of the things that are cheap to verify up front and expensive to discover in production, each with the probe that answers it. Trigger on "integrate with broker X", "the fill price is null", "why was I charged FX", "API rate limit", or before a first live trade.
---

# Verify the broker's API before you trust it

Every item below cost a real project days of debugging because it was assumed
rather than checked. Each is answerable in minutes on a paper account.

## The one probe that answers most of it

**Place the smallest possible order and sell it straight back**, then read the
fill record in full. In ten seconds and pennies of paper money you learn:

- the **real round-trip cost** (the gap between the two fill prices *is* the
  spread plus slippage - no quote feed required)
- which **fees** apply per order, and in which currency
- the **shape of the fill payload** - field names, nesting, what is absent
- how long a fill takes to become **readable in order history**
- whether the instrument code you guessed is correct

Wrap the sell in a `finally` block so no path leaves a position open, and guard
it against running outside market hours.

## Checklist

**Authentication and environments**
- Are paper and live **separate keys**, and separate hosts? Using a paper key
  against the live host usually fails with an unhelpful error.
- Do key **permissions** gate the endpoints you need? A key missing "history"
  will place orders happily and never tell you the fill price.

**Fills**
- Does the order endpoint keep returning the order after it fills, or **404**?
  Many brokers drop it from the live book within a second, which looks like
  failure if you treat 404 as an error.
- Where does the **executed price** actually live? Often only in order history,
  not the order record.
- **How long until history carries it?** Measure at the times you actually trade.
  One broker: ~16s for buys but 34-61s for orders placed a minute after the open.
  A 50s timeout lost fills on exactly one leg and looked like a mystery.

**Rate limits**
- Read `x-ratelimit-limit` / `x-ratelimit-period` from a real response rather
  than trusting documentation. One broker allowed **6 history requests per
  minute** while the obvious poll interval produced 12.
- Check whether exceeding the limit returns **429** or degrades silently.

**Money and currency**
- Is the account **multi-currency**, and does the *API* expose that? One broker
  supports per-order currency selection **in the app only** - API orders always
  settle in the account's primary currency and pay the conversion fee, with the
  foreign balance invisible in every API response.
- Which fees are **per trade** versus **per conversion**? They are charged
  separately and only one of them is avoidable.
- Does the cash figure you use for sizing come back in the **account currency**?
  Comparing it against a stake denominated in another currency is a silent bug.

**Instruments and data**
- Confirm the broker's **instrument code** format by placing a trade, not by
  guessing (`NVDA_US_EQ` vs `ITMl_EQ` vs `VWRPl_EQ`).
- If you need reference prices, check your market-data vendor **covers the venue**
  before designing around it. A US-only feed returns nothing for LSE symbols -
  and worse, a bare ticker like `ITM` may resolve to a *different* US security,
  producing confident, meaningless numbers.
- What **quantity precision** does the venue accept? Sending 8 decimals where 2
  are allowed is rejected; rounding must floor for buys (never exceed the stake)
  and sell the **live position** rather than a computed remainder.

## Report what you verified, not what you assume

Write the measured values into the code as constants with a comment naming the
date and how they were obtained. The next person - or you in three weeks - needs
to know which numbers were read from the wire and which were inherited from a
blog post.
