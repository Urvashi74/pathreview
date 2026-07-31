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

**PLAN.md link:** https://github.com/Urvashi74/pathreview/blob/fix/86-update-api-rate-limiting-header/docs/PLAN.md

**Walkthrough video (recommended):** 

**Blockers or open questions:**

## Week 9 — Solution building & PR submission

### Check-in 1 (mid-week)

**Current progress:**

Before starting implementation, I captured the baseline state of `make check` and `make test-unit` on this branch (`fix/86-update-api-rate-limiting-header`) so that any new failures introduced by my work can be distinguished from pre-existing breakage.

**`make check` — baseline: FAILING** (exit 1, stopped at the `lint` step; `format` and `typecheck` never ran)

- Runs `ruff check . → black . → mypy api/ core/ ingestion/ rag/ agent/ safety/` in sequence.
- Ruff reports **182 errors**, 86 auto-fixable with `--fix`.
- Breakdown by rule: `I001` (52, import order), `F841` (24, unused locals), `E501` (20, line length), `F401` (18, unused imports), `B008` (18, function call in default arg — mostly FastAPI `Depends()`), `B904` (11, `raise` in `except` without `from`), `B007` (10, unused loop var), `N806` (3), `B905` (3), `F821` (1), `F601` (1).
- None of the errors are in files I plan to touch (`api/middleware/rate_limit.py` doesn't exist yet; `api/main.py` and `safety/rate_limiter.py` are not in the failing set).
- **Implication:** treat `make check` as noisy on this branch. After I add my new Python files, I'll run `ruff check` scoped to just those files to confirm my own additions are clean.

**`make test-unit` — baseline: FAILING** (exit 1, 53 failed / 375 passed of 428 total, 17.88s)

- Failures span 15 pre-existing test files, largest clusters: `test_review_service.py` (13), `test_bias_detector.py` (10), `test_resume_parser.py` / `test_skill_extractor.py` / `test_pii_scrubber.py` (5 each), `test_faithfulness_checker.py` (4).
- **`test_rate_limiter.py` — all 19 tests pass.** The class I'm building on top of (`safety/rate_limiter.py::RateLimiter`) is fully green, so I can trust its behavior when writing the middleware.
- One deprecation warning: Pydantic V2 class-based `config` in `core/config.py:7` (cosmetic).
- **Implication:** the baseline test suite cannot be used as a pass/fail signal for my PR. During implementation I'll run only my new integration tests (`tests/integration/test_rate_limit_headers.py`) plus `tests/unit/test_rate_limiter.py` to verify nothing regresses.

**Sub-tasks from PLAN.md completed so far:**

- **Step 1 — Confirm the Redis client shape.** ✅ Done. Found and worked around a latent bug in `api/routes/health.py:44-49`: it constructs `redis.Redis(host=settings.redis_host, port=settings.redis_port, ...)` but neither field exists in `core/config.py` (only `redis_url` does) and neither is set in `.env`. The `AttributeError` is silently caught by the try/except at `health.py:53-56`, which is why my reproduction's `/health` reported `redis: "unhealthy"`. Decision: for the middleware I will use `redis.Redis.from_url(settings.redis_url, decode_responses=True)` — the field that actually exists — constructed once at app startup and injected. Filed the `health.py` bug as a suggested follow-up in the eventual PR description; out of scope for issue #86.
- **Step 2 — Create the middleware.** ✅ Done. Added `api/middleware/rate_limit.py` — a `RateLimitMiddleware(BaseHTTPMiddleware)` that (a) exempts `/` and `/health` (PLAN Risk #4), (b) derives an identifier as `user:<id>` → `ip:<host>` → `ip:unknown`, (c) calls the existing `RateLimiter.check_rate_limit()`, (d) short-circuits with `429 + Retry-After + X-RateLimit-Limit + X-RateLimit-Remaining:0` on rejection, and (e) attaches the two headers to the downstream response on success. Ruff-clean scoped to the new file (`ruff check api/middleware/rate_limit.py` → `All checks passed!`); the class imports cleanly.
- **Step 3 — Wire it in `api/main.py`.** ✅ Done. Constructed the Redis client once at module scope with `redis.Redis.from_url(settings.redis_url, decode_responses=True)` and passed it into a `RateLimiter` instance, then `app.add_middleware(RateLimitMiddleware, limiter=..., limit=settings.rate_limit_per_minute, window_seconds=60)`. Registered **before** `RequestIDMiddleware` in the source so that Starlette's reverse-add ordering places `RequestIDMiddleware` outermost — my rate-limit log lines now carry `request_id` in structlog context (Risk #3 decision). Verified the middleware chain at runtime: `RequestIDMiddleware` → `RateLimitMiddleware` → `CORSMiddleware` → app.

  **End-to-end verification** (Redis was reachable; flushed `rate_limit:ip:*` keys first to demonstrate the limit boundary from zero):

  ```
  $ curl -si http://localhost:8000/profiles | head -10
  HTTP/1.1 405 Method Not Allowed
  server: uvicorn
  allow: POST
  x-ratelimit-limit: 60
  x-ratelimit-remaining: 59
  x-request-id: 2d2a228e-ec9e-47d1-8919-8c33a33a333f
  ```

  Headers now present on responses. Decrement observed across 5 sequential requests (`59 → 58 → 57 → 56 → 55`). After flushing and firing 65 requests: **60 got 405** (allowed by limiter, rejected downstream because `/profiles` is POST-only) and **5 got 429** — clean boundary behavior. The 429 response:

  ```
  HTTP/1.1 429 Too Many Requests
  x-ratelimit-limit: 60
  x-ratelimit-remaining: 0
  retry-after: 60
  x-request-id: e66181d5-dd75-42ed-af37-34b1b3594bb6

  {"detail":"Rate limit exceeded"}
  ```

  Exempt paths verified: `GET /` returned `200` with **no** rate-limit headers (as designed for load-balancer health probes / root path).

  Note on lint: `api/main.py` still shows 1 `I001` (import order). That error was already present on `main` before my edit — my new import lines slotted into an already-flagged block without adding a fresh error. Zero-delta from baseline for that file. The new file `api/middleware/rate_limit.py` is lint-clean.

- **Step 5 — Integration test.** ✅ Done. Added `tests/integration/test_rate_limit_headers.py` — the first integration test in this repo (the directory previously contained only `__init__.py`). Approach: since `fakeredis` is not installed and the codebase already tests the RateLimiter–Redis interaction exhaustively in `tests/unit/test_rate_limiter.py`, I injected a small in-memory `StubLimiter` that mirrors the `(allowed, remaining)` tuple contract. This tests the middleware's actual behavior (header attachment, 429 short-circuit, exempt paths, error-response handling) hermetically with no external services and no new dependencies.

  7 tests, all passing (0.86s runtime):
  1. `test_headers_present_on_200_response` — first call to `/test` returns 200 with `X-RateLimit-Limit: 60`, `X-RateLimit-Remaining: 59`.
  2. `test_remaining_decrements_across_calls` — 3 sequential calls → remaining goes `59 → 58 → 57`.
  3. `test_returns_429_when_limit_exceeded` — with `limit=3`, the 4th call returns `429` with body `{"detail": "Rate limit exceeded"}` and headers `X-RateLimit-Limit: 3`, `X-RateLimit-Remaining: 0`, `Retry-After: 60`.
  4. `test_429_uses_configured_window_seconds_for_retry_after` — with `window_seconds=30`, the 429's `Retry-After` reflects the configured window (30, not the default 60).
  5. `test_exempt_path_has_no_ratelimit_headers` — `/health` returns 200 with no rate-limit headers, and the limiter's internal counter dict is empty (proves exempt paths do not consume budget).
  6. `test_exempt_root_path` — same for `/`.
  7. `test_headers_present_on_error_response` — a route that raises `HTTPException(500)` still gets headers attached — the middleware doesn't skip the response-decoration path for error statuses.

  Combined with existing tests: `pytest tests/unit/test_rate_limiter.py tests/integration/test_rate_limit_headers.py` → **26 passed** in 0.66s. Ruff-clean on the new file (`ruff check tests/integration/test_rate_limit_headers.py` → `All checks passed!`).

  One pre-existing warning surfaced: `StarletteDeprecationWarning: Using httpx with starlette.testclient is deprecated; install httpx2 instead.` This is a Starlette 0.x → 0.y deprecation, not caused by my code — it will fire on any `TestClient` import in this repo. Out of scope for issue #86.


**Next steps:**

- **Step 6 — Regression check.** Run `make test-integration` (my new file should be picked up cleanly) and re-run `make test-unit` to confirm zero delta vs. the recorded baseline (53 failed / 375 passed). Also run `pytest tests/unit/test_rate_limiter.py tests/integration/test_rate_limit_headers.py` to confirm the rate-limit surface stays green.
- **Step 7 — Scoped lint.** Run `ruff check` scoped to the files I changed (`api/middleware/rate_limit.py`, `api/main.py`, `tests/integration/test_rate_limit_headers.py`) — new files must be clean; `api/main.py` should still show only the pre-existing `I001` from baseline (zero-delta).
- **Step 8 — Write-up and PR.** Update Week 9 → Check-in 2: final `curl` output against the running server showing headers on 200/429, list of files touched, note the pre-existing `health.py` bug in the PR description as a suggested follow-up, and open the PR against `main`.

**Blockers:**
[Anything slowing you down? Or leave blank.]

---

### Check-in 2 (end of week)

**PR link:** [link to your submitted pull request]

**Branch:** [the branch name you worked on, e.g. `fix/123-short-description`]

**What you built:**
[1–3 sentences summarizing what your fix does and how it works]

**Tests added or updated:**
[Which test files did you touch? What do they cover?]

**Self-review confirmation:** [ ] make check passes  [ ] make test-unit passes

**Draft PR feedback received from:** [name or Slack handle, or "none"]