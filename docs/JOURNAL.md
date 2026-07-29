## Week 7 — Issue selection

**Issue link:** https://github.com/ascherj/pathreview/issues/86

**Issue title:** Add an API rate limiting header (X-RateLimit-Remaining) to responses

**Tier:** [ ] Tier 1  [x] Tier 2  [ ] Tier 3

**Problem summary:**
The issue is that right now, clients have no way to see theri current usage until a request is rejected with 429. The `RateLimiter` in `safety/rate_limiter.py` already computes remaining requests per identifier against a Redis rolling window and `core/config.py` exposes a `rate_limit_per_minute` `api/middleware/` consumes them - the middleware package only contains `auth.py` and `request_id.py`, so no rate-limit info is attached to outbound responses. Clients therefore have no way to see their current usage until a request is rejected with 429.
A successful fix would add a FastAPI middleware that runs `check_rate_limit` on each request and sets `check_rate_limit` on each request and set `X-RateLimit-Limit` and `X-RateLimit-Remaining` on every response (both allowed and 429), using the values already returned by the limiter.

**Scope fit rationale:**
This issue is a good fit for me because it is API-based and I am a backend engineer, so it plays to my existing strengths in server-side work. Beyond that, rate limiting is a topic I have specifically wanted to explore and learn about — this issue gives me a concrete, well-scoped entry point into it. The limiter itself already computes the values I need, so the work stays focused on wiring a FastAPI middleware to expose them on responses rather than designing a rate-limiting system from scratch. That keeps the scope tight while still letting me build up practical familiarity with rate-limiting headers and middleware patterns.

**Branch name:** `fix/86-update-api-rate-limiting-header`

**Setup confirmation:** [x] App runs locally at localhost:5173

**Cohort ledger:** [x] Issue added to cohort ledger

## Week 8 — Reproduction & solution planning

**Reproduction commit link:** https://github.com/Urvashi74/pathreview/commit/b6afab23dba618ae2870b5ef562a9a5ca84181a9

**Reproduction summary:**

Confirmed the gap in my local environment against the running backend (`uvicorn` on `localhost:8000`).

**Steps to reproduce:**

1. **Start the backend locally.**
   From the project root, run `make run`. The FastAPI app comes up on `http://0.0.0.0:8000` (Uvicorn logs `application_startup_completed`).

2. **Verify the server is responding.**
   ```
   $ curl -s -o /dev/null -w "HTTP %{http_code}\n" http://localhost:8000/
   HTTP 200
   ```

3. **Inspect the response headers on a normal request.**
   ```
   $ curl -si http://localhost:8000/ | head -20
   HTTP/1.1 200 OK
   date: Wed, 29 Jul 2026 01:28:08 GMT
   server: uvicorn
   content-length: 57
   content-type: application/json
   x-request-id: 72194da1-2abf-4c88-a218-886d0892e07e

   {"message":"PathReview API is running","version":"1.0.0"}
   ```
   Observed: `x-request-id` is present (proves the middleware chain runs on this path) but **no `X-RateLimit-Limit` and no `X-RateLimit-Remaining` header is returned.** Same result on `/health`.

4. **Confirm the limiter is not enforcing either — hit `/` 100 times in a row and tally status codes.**
   ```
   $ for i in $(seq 1 100); do curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8000/; done | sort | uniq -c
    100 200
   ```
   All 100 requests returned `200`. No `429` is ever emitted, which means the rate limiter is not on the request path at all — not just missing headers, but fully unwired.

5. **Locate the gap in code.**
   - `safety/rate_limiter.py:21-63` — `RateLimiter.check_rate_limit()` already returns `(allowed, remaining)`, so the values needed for the headers are computed and available.
   - `api/middleware/` — contains only `auth.py` and `request_id.py`; no rate-limit middleware exists.
   - `api/main.py:44-54` — registers `CORSMiddleware` and `RequestIDMiddleware`; `RateLimiter` is never instantiated or added to the app.
   - `grep -rn RateLimiter api/` returns no matches, confirming the API layer never touches the limiter class.

**What this proves:** the issue is real and reproducible in my local environment. The `RateLimiter` class exists in `safety/` but has never been wired into the FastAPI request lifecycle, so no `X-RateLimit-*` headers are attached to any response and no request is ever throttled. The fix will live in a new `api/middleware/rate_limit.py` (modeled on `request_id.py`) and its registration in `api/main.py`.

**PLAN.md link:** [link to PLAN.md in your fork]

**Walkthrough video (recommended):** [link to your Loom video, ≤2 min — recommended, not graded]

**Blockers or open questions:**
