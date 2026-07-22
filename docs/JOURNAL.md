## Week 7 — Issue selection

**Issue link:** https://github.com/ascherj/pathreview/issues/86

**Issue title:** Add an API rate limiting header (X-RateLimit-Remaining) to responses

**Tier:** [ ] Tier 1  [x] Tier 2  [ ] Tier 3

**Problem summary:**
The issue is that right now, clients have no way to see theri current usage until a request is rejected with 429. The `RateLimiter` in `safety/rate_limiter.py` already computes remaining requests per identifier against a Redis rolling window and `core/config.py` exposes a `rate_limit_per_minute` `api/middleware/` consumes them - the middleware package only contains `auth.py` and `request_id.py`, so no rate-limit info is attached to outbound responses. Clients therefore have no way to see their current usage until a request is rejected with 429.
A successful fix would add a FastAPI middleware that runs `check_rate_limit` on each request and sets `check_rate_limit` on each request and set `X-RateLimit-Limit` and `X-RateLimit-Remaining` on every response (both allowed and 429), using the values already returned by the limiter.

**Branch name:** `fix/86-update-api-rate-limiting-header`

**Setup confirmation:** [x] App runs locally at localhost:5173

**Cohort ledger:** [x] Issue added to cohort ledger