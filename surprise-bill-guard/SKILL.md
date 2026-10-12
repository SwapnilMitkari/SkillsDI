---
name: surprise-bill-guard
description: Find code that will run up a large cloud, API, or service bill once deployed. Use before every deploy, especially for code that calls paid APIs (including AI/LLM models), sends SMS or email, runs scheduled jobs, polls services, or scales automatically. Works across languages, including Python (Django, Flask), Java, Node.js, IBM ESQL, and front-end JavaScript.
---

# Surprise Bill Guard

Some code is cheap in testing and expensive in production: a loop that calls a paid API per item, a retry that never stops, a polling job every second, or an endpoint anyone on the internet can call to trigger paid work. This skill finds those patterns and estimates the exposure before you deploy.

## When to use

Run before every deploy, and after any change that adds an external API call, a background job, a new queue or storage use, a new public endpoint, or an autoscaling setting.

## Steps

1. **List every paid service the code touches.** Search for SDK imports, HTTP clients, and env var names. Typical categories:
   - AI and LLM APIs (OpenAI, Anthropic, Google, Mistral, Cohere, and similar)
   - Messaging: SMS, email, push, WhatsApp
   - Maps and geocoding, translation, speech, OCR, image generation
   - Pricing, quote, or market-data APIs called from a browser or a script
   - Cloud compute, serverless functions, containers, databases, object storage, egress, logging, monitoring
   - Payment and identity providers that charge per active user or per verification
   - In ESQL and App Connect: outbound HTTP nodes, SOAP or REST requests, and MQ or database nodes that call paid services

   **A call is paid when it is a real client to a paid service.** Do not decide from a comment that says "paid". Look for the SDK import, the HTTP endpoint, or the env var name. If you cannot tell, label the call "unclear" and ask.

2. **Check each call site for these cost traps.**

   | Trap | What to look for | Stack examples |
   |---|---|---|
   | Unbounded loops | Paid call inside a loop over user data, a whole table, or a list that grows. Search (case-insensitive): `while\s*\(`, `while true`, `WHILE TRUE` (ESQL), `for ... in`, `for (... : ...)` | All |
   | Repeating timers | Scheduled work that runs often: `setInterval` (JS), `scheduleAtFixedRate` or `ScheduledExecutorService` (Java), `time.sleep` inside a loop (Python), cron expressions every minute, `@Scheduled(fixedRate=...)` (Spring) | All |
   | Missing cache | Same expensive call repeated for the same input | All |
   | Retries without limits | Retry loops with no max attempts, no backoff, or retries on every error including client errors. Search: `retry`, `catch` followed by a loop, `except` followed by a loop | All |
   | Browser polling | Front-end code that calls a paid API on a timer or on every keypress with no debounce | HTML, JS, React, Vue, Angular |
   | Expensive defaults | Large model, high `max_tokens`, high resolution, or long context used by default; no cap on output length | All |
   | Public, unauthenticated endpoints | Route handlers that call a paid service with no login, no rate limit, and no quota. Search: `@app.route` (Flask), `@app.get`, `@app.post`, `@RequestMapping`, `@GetMapping`, `@PostMapping`, `path(` in Django `urls.py`, `router.` (Express), `Route::` (Laravel) | Flask, Django, Spring, Express |
   | Unpaginated queries | Fetching all rows (`.objects.all()`, `findAll()`, `SELECT *` with no `LIMIT`), or scanning whole buckets, on each request or each run | Django, Java, Node.js, ESQL |
   | Data egress | Large files or responses served to the internet without compression or CDN | All |
   | Logging payloads | Full request and response bodies logged at high volume, especially on paid services | All |
   | Unbounded scaling | Autoscaling or concurrency with no maximum; thread pools with no size limit; serverless with no concurrency cap; `min` instances set high | Java, Node.js, serverless |
   | Storage growth | Uploads, logs, or backups with no retention policy | All |
   | Test-only switches | Mocks disabled, sandbox turned off, or paid "live" mode enabled in production config | All |
   | Duplicate triggers | Event handlers or webhooks that can fire twice and charge twice (no idempotency key) | All |

3. **Estimate the exposure.** For each risky call:

   `runs per day × items per run × 30 × unit price = rough monthly cost`

   For a timer, runs per day = `86400 / interval_in_seconds` (for example, a 500 ms timer is 172,800 runs per day). For a loop with no exit, say so: the cost grows until the process stops.

   Use the provider's published price, and state the source and date. If you cannot find a price, say the estimate is unknown. Do not invent numbers. Give a range when traffic is unknown, for example "100 to 10,000 users per day."

4. **Report.** Sort by estimated exposure.

   | Risk | Location | Trigger | Estimated monthly cost | Fix |
   |---|---|---|---|---|
   | High | `jobs/sync.py:40` | Calls paid geocoding for every row in the table each run | Unbounded; about $X per 10k rows | Cache results; process only new rows |
   | High | `app.js:3` | Paid quote API every 500 ms in the browser | 172,800 calls per day per open page | Poll less often; cache on the server |
   | High | `views.py:12` | Public route sends paid SMS with no login or quota | Unbounded; anyone can trigger it | Require login; add rate limit |

5. **Recommend guardrails** (propose only, do not change accounts without approval):
   - Budget alerts and hard spending caps in the cloud or API provider console
   - Rate limits and per-user quotas on AI, messaging, and public endpoints
   - Maximum instance, thread-pool, and concurrency settings
   - Retry limits with exponential backoff
   - A kill switch (environment flag) for each paid feature

## Rules

- Mark each estimate as an estimate and show the assumptions.
- Do not change billing settings, spending caps, or account limits. Those are the user's to set.
- If the code makes no paid calls and has no scaling config, say so clearly. That is a valid result.
- Do not count a stub (a function that returns fixed data) as a paid call. Say that it is a stub and note what it will become.
