---
name: surprise-bill-guard
description: Find code that will run up a large cloud, API, or service bill once deployed. Use before every deploy, especially for code that calls paid APIs (including AI/LLM models), sends SMS or email, runs scheduled jobs, polls services, or scales automatically. Works across languages, including Python (Django, Flask), Java, Node.js, IBM ESQL, and front-end JavaScript.
---

# Surprise Bill Guard

Some code is cheap in testing and expensive in production: a loop that calls a paid API per item, a retry that never stops, a polling job every second, a busy loop with no sleep, or an endpoint anyone on the internet can call to trigger paid work. This skill finds those patterns and estimates the exposure before you deploy.

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

   **A call is paid when it is a real client to a paid service.** Do not decide from a comment that says "paid". Look for the SDK import, the HTTP endpoint, or the env var name.

   **A stub is not a paid call.** A function is a stub when its body returns fixed data and makes no network call. Say so, and note what it will become. If you cannot tell whether a call is real or a stub, label it **unclear** and say what is missing (for example, "the client module is not in the project").

   **A placeholder endpoint is scored as live.** A call whose host ends in `.example`, `.test`, or `example.com`, or that points at `localhost`, is a placeholder. Score its severity as if the host were live, because the project may ship with it, and label the report row "placeholder host". Wiring still decides Current or Latent.

   **A real call** uses a provider SDK, or an HTTP client aimed at a provider host, or reads a provider key from the environment.

2. **Check each call site for these cost traps.**

   | Trap | What to look for | Stack examples |
   |---|---|---|
   | Unbounded loops | Paid call inside a loop over user data, a whole table, or a list that grows. Search (case-insensitive): `while\s*\(`, `while true`, `WHILE TRUE` (ESQL), `for ... in`, `for (... : ...)` | All |
   | Busy loops with no sleep | `while (true)` or `while True` with no `sleep`, `await`, or blocking call inside. Runs as fast as the CPU allows, and on metered compute it costs money even when no paid call is made. | Java, Node.js, Python, serverless |
   | Repeating timers | Scheduled work that runs often: `setInterval` (JS), `scheduleAtFixedRate` or `ScheduledExecutorService` (Java), `time.sleep` inside a loop (Python), cron expressions every minute, `@Scheduled(fixedRate=...)` (Spring) | All |
   | Missing cache | Same expensive call repeated for the same input | All |
   | Retries without limits | Retry loops with no max attempts, no backoff, or retries on every error including client errors. Search: `retry`, `catch` followed by a loop, `except` followed by a loop | All |
   | Browser polling | Front-end code that calls a paid API on a timer or on every keypress with no debounce | HTML, JS, React, Vue, Angular |
   | Exposed keys in browser code | A paid API key that ships to the browser lets anyone spend the account's money. Report it here as a High, Current cost exposure (a live-prefix key in browser code, whatever the call volume), and refer it to `leaked-key-finder`. | HTML, JS |
   | Expensive defaults | Large model, high `max_tokens`, high resolution, or long context used by default; no cap on output length | All |
   | Public, unauthenticated endpoints | Route handlers that call a paid service with no login, no rate limit, and no quota. Search: `@app.route` (Flask), `@app.get`, `@app.post`, `@RequestMapping`, `@GetMapping`, `@PostMapping`, `path(` in Django `urls.py`, `router.` (Express), `Route::` (Laravel) | Flask, Django, Spring, Express |
   | Unpaginated queries | Fetching all rows (`.objects.all()`, `findAll()`, `SELECT *` with no `LIMIT`), or scanning whole buckets, on each request or each run | Django, Java, Node.js, ESQL |
   | Data egress | Large files or responses served to the internet without compression or CDN | All |
   | Logging payloads | Full request and response bodies logged at high volume, especially on paid services | All |
   | Unbounded scaling | Autoscaling or concurrency with no maximum; thread pools with no size limit; serverless with no concurrency cap; `min` instances set high | Java, Node.js, serverless |
   | Storage growth | Uploads, logs, or backups with no retention policy | All |
   | Test-only switches | Mocks disabled, sandbox turned off, or paid "live" mode enabled in production config | All |
   | Debug mode in production | `DEBUG = True`, `debug=True`, a dev Spring profile, or `npm run dev` in the production image. Not a cost trap by itself. Report it, but refer the details to `prod-parity-check`. | Django, Flask, Java, Node.js |
   | Duplicate triggers | Event handlers or webhooks that can fire twice and charge twice (no idempotency key) | All |

3. **Decide whether each risk is current or latent.**
   - **Current**: an entry point starts the code. Entry points, per stack:

     | Stack | Entry point |
     |---|---|
     | Python | A `__main__` block, a `manage.py` command, a route registered in `urls.py` or an `app.route` decorator, or a scheduler call |
     | Java | A `main` method, a `@Scheduled` method, or a `scheduleAtFixedRate` call reached from `main` |
     | Node.js | The `start` script in `package.json`, or a `require` chain from the file it names |
     | IBM ESQL | A flow whose input node connects to the node in question (check the `.msgflow` wiring) |
     | Front end | A `<script>` tag or a page that loads the file |
   - **Latent**: nothing in the project starts it. Say what would start it.
   - **Unclear**: the wiring is missing from the project (for example, a view with no `urls.py`, or a job with no scheduler). Say which file is missing. A wired project that cannot start (a missing dependency such as `flask-cors`, no `manage.py`, a jar that cannot be built) is also Unclear; name the missing piece.

4. **Estimate the exposure.** Always compute the call volume, because it does not need a price. For each risky call:

   - **Calls per day**: for a timer, `86400 / interval_in_seconds` (a 500 ms timer is 172,800 calls per day per open page or process). For a loop with no exit, say "unbounded; grows until the process stops". For a loop over data, `items per run × runs per day`.
   - **Calls per month**: calls per day × 30.

   **Default volumes when the project does not say.** State each default you use in the report.
   - Users per day: 100 to 10,000 (report both ends).
   - Browser page loads per day: 1 open page for a poll; 1,000 pages per day for a page-load call.
   - Items per run for a loop over data: the row count if the project shows it; otherwise 1,000.
   - Public endpoint requests per day: 100 to 10,000.
   - **Monthly cost**: `calls per month × unit price`. Use the provider's published price, and state the source and date. If you cannot find a price, write "unknown" in this column. Do not invent numbers. Give a range when traffic is unknown, for example "100 to 10,000 users per day".

5. **Report.** Sort current risks by exposure. List latent and unclear items in their own table below, so they are not mixed in with live risks.

   **Severity scale** (use it so the same project ranks the same way every time):
   - **High**: current, and either unbounded, or more than 100,000 calls per month at the upper default volume.
   - **Medium**: current, bounded, and more than 10,000 calls per month at the upper default volume.
   - **Low**: current and under 10,000 calls per month at the upper default volume; or latent; or unclear and bounded.
   - An unclear item with no volume bound is at least Medium.
   - Raise a Latent item to High only if the report says the project wires it up (for example, a route is added or a scheduler is registered).

   | Risk | Status | Location | Trigger | Calls per day | Calls per month | Monthly cost | Fix |
   |---|---|---|---|---|---|---|---|
   | High | Current | `jobs/sync.py:40` | Calls paid geocoding for every row in the table each run | Unbounded per run | Unbounded | Unknown (no price) | Cache results; process only new rows |
   | High | Current | `app.js:3` | Paid quote API every 500 ms in the browser | 172,800 per open page | about 5.2 million per open page | Unknown (no price) | Poll less often; cache on the server |
   | High | Current | `views.py:12` | Public route sends paid SMS with no login or quota | Unbounded | Unbounded | Unknown (no price) | Require login; add rate limit |
   | Latent | Latent | `worker.py:3` | Loop sends SMS per user; no file in the project starts it | Would be 172,800 per user | Would be 5.2 million per user | Unknown (no price) | Do not start without a cap |
   | Unclear | Unclear | `shop/views.py` | View is never routed (no `urls.py` in the project) | Unknown | Unknown | Unknown | Add the URL config, or confirm the view is unused |

6. **Recommend guardrails** (propose only, do not change accounts without approval):
   - Budget alerts and hard spending caps in the cloud or API provider console
   - Rate limits and per-user quotas on AI, messaging, and public endpoints
   - Maximum instance, thread-pool, and concurrency settings
   - Retry limits with exponential backoff
   - A kill switch (environment flag) for each paid feature

## Rules

- Mark each estimate as an estimate and show the assumptions.
- Always report call volumes, even when the price is unknown.
- Do not change billing settings, spending caps, or account limits. Those are the user's to set.
- If the code makes no paid calls and has no scaling config, say so clearly. That is a valid result.
- Do not count a stub as a paid call. Say that it is a stub and note what it will become.
- A real-looking live secret on the server with no paid call is not a cost risk. Note it and refer it to `leaked-key-finder`.
- Do not look up prices online unless the user asks. Use prices the user provides or a price list that is in the project.
