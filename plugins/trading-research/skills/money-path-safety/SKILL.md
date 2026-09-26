---
name: money-path-safety
description: Use when writing or reviewing code that places real orders, moves funds, or triggers any irreversible external action on a schedule. Covers idempotency against non-idempotent endpoints, recovery paths, hard stops, dry runs that exercise the guards, and self-healing records. Trigger on "automate trading", "place an order from a Lambda/cron", "duplicate order", "the position is stuck", or reviewing anything that spends money unattended.
---

# Patterns for code that spends money while you sleep

## The hazard

Most broker order endpoints are **not idempotent**: the same request sent twice
creates two real orders. Meanwhile every layer you build on retries by default -
Lambda async invocation, EventBridge targets, HTTP clients. The defaults are
wrong for this domain and must be switched off deliberately.

## Non-negotiables

**Disable every automatic retry on the path that places orders.** Lambda invoke
config and scheduler target both set to zero attempts. The HTTP client must never
retry a non-GET, even on a transport error - a POST that timed out may still have
been received. Surface it instead.

**Claim before you trade.** Take an atomic conditional write (DynamoDB
`attribute_not_exists`, a unique index, a lock row) *before* the order request.
A second invocation loses the race and exits. Without this, one duplicated
schedule event is a duplicated position.

**Two independent stops.** A time window enforced by the scheduler itself, and a
counter checked in code. Split the counter by dry-run/live so the dry run
exercises the stop that decides when real money stops being spent - otherwise
that guard is the one thing never tested.

**A recovery path that trades without the claim.** When a claim is stuck, the
normal path can never retry, and the position sits open. Provide an explicit
unwind: gated by a confirmation token, recording under its own key so it never
overwrites the record it is rescuing, and never touching the session counter - an
unwound attempt is void, not completed.

**Make the dry run traverse everything.** It should place no order but exercise
every guard, write every record, and advance its own counter. A dry run that
skips the interesting paths proves nothing.

## Records: assume capture will fail

Order placement and record-keeping are separate failures. Ours succeeded at
trading and failed at recording, repeatedly, and the numbers only survived
because of one pattern:

**Make the reconciler self-healing.** Store the `order_id` at placement time,
always. Then a later pass can re-fetch any missing fill by that id. In this
project the recovery path silently rescued three sessions whose fills were null
at trade time.

**Never stamp a record complete while data is missing.** If your reconciler skips
rows marked done at the current schema version, then writing "done" with a null
price freezes that gap forever. Use a distinct `PARTIAL` state, and version the
reconciler so a metric change re-derives history in place.

**Bound the whole operation inside its runtime.** If a leg can wait longer than
the function's timeout, it will eventually place an order and record *nothing* -
strictly worse than a null fill, because there is no id left to heal from. Keep
an explicit deadline inside the wait, and a timeout comfortably above it.

## Verification discipline

- **Check the records, not the exit code.** Every failure here returned HTTP 200.
- **Reproduce the failing conditions exactly** - same instrument, same minute -
  or a passing probe proves nothing about the failure.
- **Verify the artefact the run actually produced**, by name. A same-day rerun
  that overwrites a key, or a date rollover that creates a new one, will
  otherwise have you inspecting yesterday's output and drawing conclusions.
- **Confirm a fix on real traffic before calling it fixed**, and say how many
  occurrences you have seen. One clean run after an intermittent bug is
  suggestive, not proof.
