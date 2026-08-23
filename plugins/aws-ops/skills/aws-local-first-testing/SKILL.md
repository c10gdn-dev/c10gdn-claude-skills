---
name: aws-local-first-testing
description: Use when scaffolding tests for a new AWS-backed project, or when asked to make a codebase testable without live AWS credentials or cost. Sets up moto for AWS service mocking, respx/httpx-mock for external HTTP, dev-mode fallbacks for side-effecting integrations (e.g. writing emails to disk instead of sending), and fixture-based tests for browser automation. Trigger on "how do we test this without hitting AWS", "mock DynamoDB/SSM/Lambda locally", or when setting up a new AWS project's test suite.
---

# AWS local-first testing strategy

## Goal

The large majority of an AWS-backed app should be testable with zero AWS cost and
zero live credentials. Only true end-to-end flows against real external systems
(a real third-party login, a real paid API) should require live credentials, and
those should be a small, clearly-marked minority of the suite.

## Test levels

| Level | Tool | What it covers | AWS cost |
|---|---|---|---|
| Unit | pytest | Pure logic, no I/O | $0 |
| Integration | pytest + `moto[all]` | DynamoDB/SSM/S3/etc. calls against an in-process mock | $0 |
| HTTP-boundary | pytest + `respx` (or equivalent for the HTTP client in use) | External APIs (email providers, FX rate APIs, third-party services) | $0 |
| Browser | pytest + Playwright + saved HTML fixtures | Scrape/extract logic against a static snapshot of the real page | $0 |
| E2E | real credentials, run sparingly, marked separately | Auth flows and anything that can't be faithfully mocked | Real cost |

## Patterns

**AWS service mocking — `moto[all]`.** In-process, no Docker/LocalStack container
needed. A `conftest.py` fixture that spins up a mocked DynamoDB table (or SSM
parameters) per test session covers the large majority of AWS-touching code.

**External HTTP — mock at the client library, not the network.** `respx` for
`httpx`, or the equivalent interceptor for whatever HTTP client is in use. Assert on
the outgoing request (payload, headers, method) as well as stubbing the response —
catches "we're sending the wrong currency param" bugs that a response-only mock
would miss.

**Side-effecting integrations — a dev-mode short circuit, not a mock in every
test.** For things like outbound email: gate on an `ENVIRONMENT=development` check
and write the rendered output to a local `logs/emails/` (or similar) directory as a
plain file instead of calling the real provider. This gives free, real, visually
inspectable output during manual local development, on top of (not instead of) unit
tests that assert on the payload via the HTTP-boundary mock above.

**Browser/scrape logic — saved HTML fixtures, not live page hits.** Capture a
snapshot of each page state that matters (logged-out, logged-in, empty-results,
populated-results, error state) into `tests/fixtures/html/`, redact any personal
data before committing, and load via `page.goto("file://...")` or
`page.set_content()`. Keeps scrape/extraction-logic tests fast, deterministic, and
independent of the live site's uptime or bot protection.

**Third-party auth/2FA flows (e.g. polling an email inbox for a code) — canned JSON
response fixtures.** Record a "no messages yet" response and a "message present"
response, and have the fixture serve the former for the first N calls before
switching to the latter, to exercise the real polling/retry loop without a live
wait.

## Directory layout

```
tests/
├── conftest.py                     # moto table fixture, mock SSM client, etc.
├── fixtures/
│   ├── html/                       # saved page snapshots per state
│   ├── api_responses/              # canned JSON for polled/external APIs
│   └── mock_server.py              # optional: minimal FastAPI mock of a multi-step flow
```

## What NOT to bother mocking

Orchestration glue (e.g. a workflow/state-machine layer implemented as plain
function calls rather than an actual cloud state machine) doesn't need AWS-service
mocking at all — call the orchestrator function directly and mock only the
individual steps it calls (scrape, store, notify) as plain Python mocks. Don't spin
up a mocked Step Functions/EventBridge for logic that's really just a Python
function calling other Python functions.
