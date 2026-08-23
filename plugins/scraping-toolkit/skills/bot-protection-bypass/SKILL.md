---
name: bot-protection-bypass
description: Use when a scraping target returns 403s or CAPTCHA challenges from a commercial bot-detection vendor (DataDome, PerimeterX, Akamai Bot Manager, Cloudflare, Imperva). Diagnoses whether the block is a hard IP ban, a TLS/JA3 fingerprint mismatch, or a genuinely solvable challenge, and covers wiring up a CAPTCHA-solving service correctly including proxy IP hygiene and cookie-to-IP binding. Trigger on "blocked scraping X", "getting 403 from an API that works fine in a real browser", "DataDome/Cloudflare challenge", "CAPTCHA solver keeps returning unsolvable".
---

# Bot-protection bypass (DataDome and similar vendors)

## What these systems evaluate

Commercial bot-detection (DataDome is the concrete case this was written from, but
PerimeterX/Akamai/Cloudflare work on the same principles) scores every request on:

- IP reputation (datacenter vs residential, known bot ranges, abuse history)
- TLS/JA3 fingerprint (cipher suites and extension order in the TLS handshake)
- HTTP/2 fingerprint (header order, pseudo-header order, SETTINGS frames)
- Behavioural signals (request rate, header consistency, cookie state)

When suspicious, it returns a block response (DataDome: `403` with a challenge URL
pointing to `geo.captcha-delivery.com`) instead of the real payload.

## Step 1 — classify the block before doing anything else

For DataDome specifically, the `t=` query param on the challenge URL tells you what
you're dealing with:

| `t=` value | Meaning | Solvable? |
|---|---|---|
| `bv` (Bot Visitor) | Hard IP ban — your IP is on a global blocklist (this includes essentially all AWS/GCP/Azure/datacenter ranges). No challenge is even presented. | No — solving services reject with a bad-parameters error |
| `fe` (Force Enhanced) | Suspicious but not banned. May show a solvable slider puzzle, or may run silent JS fingerprinting with no visible challenge (`noPuzzle: true` in the page config) | Depends — see step 4 |
| `nc` (New Captcha) | Standard visual slider puzzle | Yes |

Other vendors have their own equivalent signal (Cloudflare's challenge page type,
Akamai's `_abck` cookie state) — the classification step matters regardless of
vendor: don't invest in CAPTCHA-solving integration until you know a challenge is
actually being presented rather than a silent ban.

## Step 2 — check whether the TLS fingerprint is the problem

This is the single highest-value thing to test first if you're seeing a hard ban
(`t=bv`) even through a clean proxy IP. Python's `ssl` module — used by `httpx`,
`requests`, and (surprisingly) `curl_cffi` despite its Chrome-impersonation claims —
produces a handshake fingerprint that DataDome specifically flags. `pycurl` (Python
bindings to libcurl's native TLS stack) and plain `curl` do not get flagged the same
way.

Test matrix approach — hit the target through the *same* proxy with each client and
compare the `t=` response:

| Client | Typical result through a clean proxy |
|---|---|
| `curl` (libcurl) | Usually passes / gets a real challenge |
| `pycurl` | Usually passes / gets a real challenge |
| `curl_cffi` (Chrome TLS impersonation) | Often still banned — vendor has fingerprinted it specifically |
| `httpx` / `requests` (Python ssl) | Often banned |

If switching the HTTP client from `httpx`/`requests` to `pycurl` turns a `t=bv` into
a `t=fe`/`t=nc`, that confirms TLS fingerprinting was the actual blocker, not the
proxy.

## Step 3 — proxy requirements

1. **Residential or ISP-grade IP.** Datacenter ranges (AWS, GCP, Azure,
   DigitalOcean, Hetzner, IBM SoftLayer, etc.) are broadly blocklisted. "Static
   residential" branding from a proxy vendor doesn't guarantee a non-datacenter
   exit — verify the actual exit IP against `api.ipify.org` (or an IP-reputation
   lookup) before trusting a proxy.
2. **Credential-based auth (username:password), not IP-whitelisting.** A
   CAPTCHA-solving service connects to your proxy from its own datacenter IPs to
   perform the solve — if your proxy only accepts pre-whitelisted source IPs, the
   solving service can't reach it. Quick test: `curl -x host:port https://api.ipify.org`
   with no credentials — `407 Proxy Authentication Required` means it accepts
   credential auth from any source; a bare connection refusal means IP-whitelist
   only (won't work with a third-party solver).
3. **Test every candidate proxy against the actual target before committing to
   one.** Proxy pool quality is inconsistent even within one vendor/tier — expect
   to throw away a meaningful fraction of a purchased pool.

## Step 4 — the "no visible challenge" trap (`noPuzzle: true` equivalents)

Some configurations run pure JS fingerprinting (canvas, WebGL, AudioContext,
navigator properties, mouse/touch behaviour) with **no interactive element
rendered at all**. A real browser passes this silently; a CAPTCHA-solving worker
(human or AI) sees a blank page and returns "unsolvable." This makes the target
bypass-proof via standard CAPTCHA services, full stop — the only options are a real
browser with convincing fingerprints, or a managed browser-based scraping service.

**Important:** this is a site-level config, not a permanent property of the target,
and it can change at any time. If blocked by this, it's often worth periodically
re-testing rather than immediately building a full browser-based bypass — the
vendor or site owner may revert to a solvable slider without warning.

## Step 5 — the cookie-to-IP binding constraint

After a challenge is solved, the vendor issues a session cookie that is
cryptographically bound to the IP that solved it. The entire chain — initial
blocked request, the solve itself, and the retried request with the cookie — must
exit from the **same IP**:

```
1. request → proxy (IP X) → target → 403 challenge
2. solving service → proxy (IP X) → challenge page → solved cookie
3. request → proxy (IP X) → target + cookie → 200 OK
```

If step 2 doesn't route through the identical proxy/IP as steps 1 and 3, the cookie
will be rejected on replay even though the solve itself succeeded.

## Solving services

- **2Captcha / CapSolver** — task types like `DataDomeSliderTask`
  (`DatadomeSliderTask` on CapSolver) handle real slider puzzles. Neither works
  against the silent-fingerprinting case (step 4) or a hard IP ban. Pass proxy
  credentials as part of the task so the solve happens from your chosen IP. Cost is
  roughly $2–3 per 1,000 solves.
- **Managed browser-based scraping services** (ScrapFly, ZenRows, Bright Data Web
  Unlocker) — handle the vendor natively including the silent-fingerprinting case,
  at meaningfully higher cost (~$25–100+/mo for moderate volume). Worth it when the
  target uses `noPuzzle`-style protection and a DIY bypass isn't worth the
  engineering time.

## Checklist for a new target

1. Make one request, note the block type/challenge param.
2. Hard ban even through a clean proxy? Try `pycurl`/libcurl instead of a
   Python-ssl-based client before blaming the proxy.
3. Challenge page shows no interactive element / config flags silent fingerprinting?
   Standard CAPTCHA services won't help — evaluate a managed service or a real
   browser instead.
4. Solvable challenge present? Verify the proxy accepts credential auth from
   external IPs before wiring up the solving service.
5. Ensure the retry request reuses the exact proxy/IP used for the solve.
6. If it stops working later, re-check the challenge type before assuming your
   bypass broke — the site's protection config may simply have changed.
