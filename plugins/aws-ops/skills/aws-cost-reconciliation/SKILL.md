---
name: aws-cost-reconciliation
description: Use when asked to check actual AWS spend against estimates, "what is this actually costing us", or to refresh a project's cost-estimate documentation with real billing numbers after a full billing cycle. Walks through pulling a per-service cost breakdown and reconciling it against the original Terraform/architecture-time estimate.
---

# AWS cost reconciliation

## When to run this

- After the first full calendar month of a new deploy, once real usage data exists.
- Whenever a project's cost doc still shows launch-time estimates and someone asks
  "is this actually cheap."
- Periodically (quarterly) for anything long-running, since AWS pricing and free-tier
  eligibility can shift.

## Steps

1. **Pull the real numbers.** AWS Console → Billing and Cost Management → Cost
   Explorer, grouped by Service, filtered to the relevant month. If resources are
   tagged (e.g. `Project = ticket-monitor`), filter by tag instead of eyeballing —
   far more reliable once an account runs more than one project.
2. **Line up against the original estimate table.** Go component by component
   (compute, storage, data transfer, third-party APIs billed outside AWS) rather
   than comparing only the totals — the total can look "about right" while
   individual line items are wildly off in opposite directions.
3. **Explain the deltas, don't just report them.** Common causes worth checking
   before writing the new number down:
   - Something assumed to be paid is actually inside the always-free tier (Lambda's
     1M requests/mo, DynamoDB on-demand at low volume, S3 minimal storage).
   - An always-on resource (a small EC2 instance, a NAT gateway) dominates cost far
     more than batch/on-demand compute — flag these as the first place to look if
     the bill needs to come down further.
   - Third-party costs (CAPTCHA solving, proxy services, SaaS email) are billed
     outside AWS and won't show in Cost Explorer at all — pull those from the
     vendor's own billing page and add them to the total explicitly.
4. **Update the doc, don't just report in chat.** Replace the estimate table with
   an "Actual" column (or replace it outright, labeled with the billing month), and
   add one line explaining *why* the original estimate was off — future estimates on
   the next project should learn from the direction of the error (usually: too
   pessimistic on free-tier coverage), not just get a fresh guess.

## Output shape

A short table: component → actual $ → one-line note on why it landed where it did,
plus a total. Call out the single largest line item explicitly — it's usually the
one lever worth pulling if cost needs to drop further (e.g. an always-on proxy
instance vs. batch compute that only runs minutes per day).
