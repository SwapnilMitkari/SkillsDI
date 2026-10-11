---
name: surprise-bill-guard
description: Find code that will run up a large cloud, API, or service bill once deployed. Use before every deploy, especially for code that calls paid APIs (including AI/LLM models), sends SMS or email, runs scheduled jobs, or scales automatically.
---

# Surprise Bill Guard

Some code is cheap in testing and expensive in production: a loop that calls a paid API per item, a retry that never stops, a polling job every second. This skill finds those patterns and estimates the exposure before you deploy.

## When to use

Run before every deploy, and after any change that adds an external API call, a background job, a new queue or storage use, or an autoscaling setting.

## Steps

1. **List every paid service the code touches.** Search for SDK imports, HTTP clients, and env var names. Typical categories:
   - AI and LLM APIs (OpenAI, Anthropic, Google, Mistral, Cohere, and similar)
   - Messaging: SMS, email, push, WhatsApp
   - Maps and geocoding, translation, speech, OCR, image generation
   - Cloud compute, serverless functions, containers, databases, object storage, egress, logging, monitoring
   - Payment and identity providers that charge per active user or per verification

2. **Check each call site for these cost traps.**

   | Trap | What to look for |
   |---|---|
   | Unbounded loops | Paid call inside a loop over user data, a whole table, or a list that grows |
   | Missing cache | Same expensive call repeated for the same input |
   | Retries without limits | Retry loops with no max attempts, no backoff, or retries on every error including client errors |
   | Too-frequent polling | Timers, cron, `setInterval`, `sleep` loops, or webhooks that poll more often than needed |
   | Expensive defaults | Large model, high `max_tokens`, high resolution, or long context used by default; no cap on output length |
   | No per-user limits | AI or messaging endpoints with no rate limit, quota, or auth, so anyone can trigger paid calls |
   | Unpaginated queries | Fetching all rows or all objects, or scanning whole buckets, on each request |
   | Data egress | Large files or responses served to the internet without compression or CDN |
   | Logging payloads | Full request and response bodies logged at high volume, especially on paid services |
   | Unbounded scaling | Autoscaling or concurrency with no maximum; serverless with no concurrency cap; `min` instances set high |
   | Storage growth | Uploads, logs, or backups with no retention policy |
   | Test-only switches | Mocks disabled, sandbox turned off, or paid "live" mode enabled in production config |
   | Duplicate triggers | Event handlers or webhooks that can fire twice and charge twice (no idempotency key) |

3. **Estimate the exposure.** For each risky call:

   `calls per request × requests per day × 30 × unit price = rough monthly cost`

   Use the provider's published price, and state the source and date. If you cannot find a price, say the estimate is unknown. Do not invent numbers. Give a range when traffic is unknown, for example "100 to 10,000 users per day."

4. **Report.** Sort by estimated exposure.

   | Risk | Location | Trigger | Estimated monthly cost | Fix |
   |---|---|---|---|---|
   | High | `jobs/sync.py:40` | Calls paid geocoding for every row in the table each run | Unbounded; about $X per 10k rows | Cache results; process only new rows |

5. **Recommend guardrails** (propose only, do not change accounts without approval):
   - Budget alerts and hard spending caps in the cloud or API provider console
   - Rate limits and per-user quotas on AI and messaging endpoints
   - Maximum instance and concurrency settings
   - Retry limits with exponential backoff
   - A kill switch (environment flag) for each paid feature

## Rules

- Mark each estimate as an estimate and show the assumptions.
- Do not change billing settings, spending caps, or account limits. Those are the user's to set.
- If the code makes no paid calls and has no scaling config, say so clearly. That is a valid result.
