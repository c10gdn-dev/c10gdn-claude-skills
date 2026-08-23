---
name: multi-source-scraper-architecture
description: Use when designing or extending a scraper system that pulls comparable data from multiple sources with different auth models and data shapes (authenticated vs public, browser-driven vs direct API). Covers a shared base abstraction, keeping per-source data models separate rather than forcing artificial commonality, and a checklist for cleanly bolting on a new source. Trigger on "add another data source to this scraper", "we need to pull the same data from 2+ sites", or when the third similar-but-different source is about to make a single hardcoded scraper unworkable.
---

# Multi-source scraper architecture

## Core idea

Sources that conceptually produce "the same kind of data" (e.g. ticket listings,
product prices) often differ in two independent ways that shouldn't be conflated:

- **Auth model**: none (public page) vs. login + 2FA vs. API key.
- **Fetch mechanism**: full browser automation (Playwright) vs. a direct internal
  JSON API call (no browser needed at all) vs. a browser-driven flow that hits
  bot-protection and needs a CAPTCHA-solving detour.

Design for these as orthogonal axes, not a single "scraper type" enum — a source
that's public *and* API-driven should look nothing like a source that's public *and*
browser-driven in the code, even though both are "no auth."

## Structure

```
src/browser/
├── base_scraper.py       # shared lifecycle only: browser launch/teardown, retry, logging
├── source_a/
│   ├── scraper.py         # auth (if any) + fetch, source-specific
│   └── extractors.py      # raw page/response -> structured records
├── source_b/
│   ├── scraper.py
│   └── extractors.py
└── source_c/
    ├── scraper.py          # e.g. httpx/pycurl API client — no browser at all
    └── extractors.py
```

The shared base should carry only what's *actually* common across every source
(e.g. Playwright context lifecycle, structured logging, a common retry/backoff
helper) — resist the urge to push source-specific concerns (auth, CAPTCHA
handling, pagination) up into it. A source with no browser involvement at all
(pure API fetch) legitimately doesn't inherit the Playwright base — don't force it
to just for interface consistency.

## Data models: separate by default, share only when the shape genuinely matches

Two sources that are conceptually "the same kind of thing" often are not the same
*shape* of data. E.g. one source might expose only an aggregated price range per
category while another exposes individual line-item listings with quantity — those
need different Pydantic/dataclass models, not one model with optional fields for
whichever source doesn't have that data. Force a shared model only when two sources
are *actually* identical in shape (in which case a single model with a `source`
discriminator field, or a light type alias, is the right amount of sharing — no more).

Persisted records (DB rows, files) should carry a per-source prefix or discriminator
in whatever their key/identifier scheme is, so per-source and cross-source queries
are both possible without a schema migration when a new source is added later.

## Checklist for adding a new source

1. Confirm auth model and fetch mechanism (the two axes above) before writing any
   code — this decides whether it extends the browser base at all.
2. Define its data model — start from scratch, don't force-fit an existing source's
   model; converge later only if the shape turns out identical.
3. Build the extractor: raw response/page → structured records. Keep this pure
   (no I/O) and unit-testable against saved fixtures.
4. Build the scraper: fetch + auth (if any) + call the extractor. Handle
   source-specific failure modes here (bot-protection 403s, auth timeouts) with
   graceful degradation (return empty results, don't crash the whole run) if this
   source going down shouldn't block the others.
5. Wire into the orchestrator's parallel fan-out step, not sequentially after the
   others, so one slow/failing source doesn't delay the rest.
6. Add the new source's discriminator/prefix to storage, summary generation, and
   notification templates — these three are the usual places a new source gets
   half-wired-in and then silently missing from one of them.
7. Document the auth requirements (or lack thereof) and any bot-protection
   specifics up front — see the `bot-protection-bypass` skill if this source needs
   it.
