# API Design — Interview Simulator

**Instructions:** Set a 35-minute timer per scenario. Attempt completely before reading guidance. A senior interviewer expects naming conventions, status code justification, versioning strategy, idempotency handling, and failure mode awareness.

---

## Scenario 1: Design the Stripe Payments API

**Prompt:**
> "Design the REST API for a payment processing service like Stripe. It must support: creating a payment intent, confirming a payment, issuing refunds, and listing a customer's payment history. The API will be consumed by thousands of third-party developers building e-commerce platforms. Walk me through your resource model, HTTP methods, status codes, versioning strategy, idempotency design, and how you handle retries after network failures."

*35 minutes. Start now.*

---

### Interviewer Follow-ups
1. "A developer retries a POST /payment-intents after a timeout and accidentally charges the customer twice. How does your API prevent this?"
2. "You need to add a new field `currency_code` to the PaymentIntent response 6 months from now. Is that a breaking change? What if you need to rename `amount` to `amount_cents`?"
3. "Your API has 10,000 developers on v1. You need to ship v2 with breaking schema changes. What's your migration plan?"
4. "A payment was confirmed but the confirmation response was never received by the client. What does the client do?"

---

### Guidance (Read After Attempting)

**Resource Model:**
```
POST   /v1/payment-intents                  Create payment intent (→ 201)
GET    /v1/payment-intents/{id}             Fetch status (→ 200)
POST   /v1/payment-intents/{id}/confirm     Confirm + charge (→ 200)
POST   /v1/payment-intents/{id}/cancel      Cancel intent (→ 200)
POST   /v1/refunds                          Issue a refund (→ 201)
GET    /v1/customers/{id}/payment-intents   List history (cursor-paginated)
```

**Status codes:**
- `POST /payment-intents` → 201 + `Location: /v1/payment-intents/pi_123`
- `POST /{id}/confirm` → 200 (not 201 — no new resource created)
- Payment already confirmed, re-confirm → 409 Conflict
- Card declined → 402 Payment Required (special case)
- Rate limited → 429 + Retry-After

**Idempotency design (Stripe's actual approach):**
Every POST requires an `Idempotency-Key` header.
- Server stores `(key → {status_code, response_body})` in Redis, TTL 24h
- Same key + same endpoint → return stored response, `Idempotency-Replayed: true`
- Same key + different endpoint or body → 422 Unprocessable Entity

```
POST /v1/payment-intents
Idempotency-Key: pi_create_order_99_attempt_1
{ "amount": 9999, "customer_id": "cus_alice" }
```

**Versioning:**
URI path versioning (`/v1/`). Stripe has maintained `/v1/` since 2011 and uses a dated **API version** pinned per-customer in the API key metadata (e.g., `api-version: 2024-04-15`). Additive non-breaking changes are released without version bumps; breaking changes bump the pinned API version.

**Migration plan for v2:**
1. Ship v2 endpoint alongside v1 (both live simultaneously)
2. Add `Deprecation` + `Sunset` headers to all v1 responses
3. Email affected developers with changelog and migration guide
4. Log v1 usage per API key to track adoption
5. 12-month deprecation window → enforce sunset → return 410 on v1 after sunset date

**Retry after lost confirmation:**
Client uses `GET /v1/payment-intents/{id}` to check current status. If status is `succeeded` → payment went through, don't retry POST confirm. This is the "check before retry" pattern for non-idempotent operations.

---

## Scenario 2: Design the Twitter/X Timeline REST API

**Prompt:**
> "Design the REST API that powers the Twitter home timeline. Users can post tweets, fetch their home timeline (tweets from people they follow), like a tweet, and search tweets by keyword. Millions of mobile and web clients hit this API. Cover: resource modelling, HTTP methods, pagination approach, caching strategy, and how you'd handle a client that needs very different data shapes on mobile vs web."

*35 minutes. Start now.*

---

### Interviewer Follow-ups
1. "A user likes a tweet twice (double-tap). What does your API return on the second like?"
2. "The mobile app only needs tweet text and author avatar. The web app needs engagement counts, media, quoted tweets, and ad slots. How does your API serve both efficiently?"
3. "A tweet has been deleted but cached copies exist in CDN. What do you do?"
4. "You're designing the timeline endpoint. How do you paginate 3.2 billion tweets in real time while new ones are being added every millisecond?"

---

### Guidance (Read After Attempting)

**Resource Model:**
```
POST   /v1/tweets                           Post a tweet (→ 201)
GET    /v1/tweets/{tweet_id}               Fetch single tweet (→ 200; cacheable)
DELETE /v1/tweets/{tweet_id}               Delete tweet (→ 204)
GET    /v1/users/{user_id}/timeline        Home timeline (cursor-paginated)
POST   /v1/tweets/{tweet_id}/likes         Like a tweet (→ 200 or 204)
DELETE /v1/tweets/{tweet_id}/likes/{user}  Unlike (→ 204; idempotent)
GET    /v1/search/tweets?q={query}          Search (→ 200, cursor-paginated)
```

**Liking twice:**
First like → 200 (or 204). Second like (already liked) → 200 with current counts (not 4xx). The operation is idempotent at the business level — liking a tweet you already liked has no effect. Return the tweet's current like count in the response so the client can sync state.

**Mobile vs web — field selection:**
```
GET /v1/tweets/{id}?fields=text,author.avatar          ← mobile (3 fields)
GET /v1/tweets/{id}                                    ← web (full payload)
GET /v1/tweets/{id}?expand=media,quoted_tweet,metrics  ← explicit expansion
```
Field selection via `?fields=` + expansion via `?expand=` is Twitter's actual approach. The server defaults to a minimal set; clients opt-in to more.

**Cache invalidation on delete:**
1. Return `204` on delete.
2. Purge CDN cache via CDN API (Cloudflare/Akamai purge by URL).
3. Set short TTL on individual tweet responses (60s) to limit stale window.
4. Use `Cache-Control: no-store` for currently-authenticated user's own tweets (always fresh).
5. Downstream: publish `tweet.deleted` event → Elasticsearch removes document → search index updated.

**Timeline Pagination:**
Offset/limit fails at scale and with real-time insertion. Use cursor pagination based on tweet snowflake ID (which is time-sortable):
```
GET /v1/users/42/timeline?count=20                  → first page, newest tweets
GET /v1/users/42/timeline?max_id=1842...&count=20   → older tweets (scroll down)
GET /v1/users/42/timeline?since_id=1843...&count=20 → newer tweets (pull to refresh)
```
`max_id` for older content, `since_id` for polling new content. Snowflake IDs sort chronologically without a separate `ORDER BY created_at` clause, making the index seek efficient.

---

## Scenario 3: Design the GitHub REST API for Repository Management

**Prompt:**
> "Design the API for a code hosting platform like GitHub. It must support managing repositories (create, list, delete), getting file contents, creating issues, and listing commits. The API is used by the GitHub CLI, VS Code extensions, CI/CD pipelines (automated machine-to-machine), and casual browser users. Cover: authentication approach per use case, versioning, rate limiting design, and idempotency for issue creation."

*35 minutes. Start now.*

---

### Interviewer Follow-ups
1. "A CI pipeline calls `POST /repos/{owner}/{repo}/issues` to file a build failure report. The network times out. The pipeline retries 3 times. There are now 3 duplicate issues. How do you prevent this?"
2. "The GitHub CLI needs to list all 5,000 commits in a repository. How does pagination work?"
3. "A public repo's README is fetched 100 million times per day. How do you handle this without hammering your database?"
4. "You have three client types (browser user, CLI tool, CI pipeline). Each has different rate limit needs. How do you design the rate limiting?"

---

### Guidance (Read After Attempting)

**Resource Model:**
```
POST   /v1/user/repos                           Create repo (→ 201)
GET    /v1/repos/{owner}/{repo}                 Get repo (→ 200; public = cacheable)
DELETE /v1/repos/{owner}/{repo}                 Delete repo (→ 204)
GET    /v1/repos/{owner}/{repo}/contents/{path} Get file (→ 200; heavy caching)
GET    /v1/repos/{owner}/{repo}/commits         List commits (cursor-paginated)
POST   /v1/repos/{owner}/{repo}/issues          Create issue (→ 201)
GET    /v1/repos/{owner}/{repo}/issues          List issues (cursor-paginated)
```

**Authentication per client type:**

| Client | Auth Method | Why |
|--------|------------|-----|
| Browser user | OAuth2 PKCE flow → short-lived JWT | Interactive, user context |
| CLI tool | Personal Access Token (PAT) scoped to repos | Long-lived, no browser |
| CI/CD pipeline | GitHub App installation token (scoped, auto-rotated) | Machine-to-machine, short-lived |
| Third-party app | OAuth2 Authorization Code flow | Delegate with scopes |

Never pass tokens in URLs. Always `Authorization: Bearer {token}` header.

**Duplicate issue prevention (CI pipeline retry):**
Option A: Client-side idempotency key
```
POST /v1/repos/owner/repo/issues
Idempotency-Key: ci-job-{job_id}-build-failure
{ "title": "Build failed", "body": "..." }
```
Server stores key → issue_number mapping. Retry returns existing issue.

Option B: Natural deduplication in application logic
```
# CI pipeline checks before creating:
existing = GET /v1/repos/owner/repo/issues?labels=ci-failure&state=open&q="Build+#{job_id}"
if exists: comment on existing issue
else: POST create new issue
```
Option A is cleaner — put dedup logic in the server, not the client.

**Commit list pagination:**
Commits are naturally cursor-paginated via SHA + `until`:
```
GET /v1/repos/owner/repo/commits?per_page=100&sha={last_sha_from_prev_page}
```
Each page's `Link` header contains `rel="next"` URL with the next cursor pre-built. 5,000 commits = 50 pages at 100/page. Range: milliseconds per page.

**README caching at 100M requests/day:**
- `Cache-Control: public, max-age=60` on public repo content
- CDN (Cloudflare / Fastly) caches by URL — file contents are URL-addressed
- ETag based on file SHA: `ETag: "sha256:{content_hash}"`
- Client sends `If-None-Match: {etag}` → 304 Not Modified if unchanged (zero body transfer)
- 100M requests/day ÷ 86,400 seconds ≈ 1,157 requests/sec — almost entirely CDN-served, ~1% cache misses hit origin

**Rate limiting per client type:**

```
Browser user (OAuth2):   5,000 requests/hour per authenticated user
Personal Access Token:   5,000 requests/hour per token
Unauthenticated:         60 requests/hour per IP
CI/CD App token:         15,000 requests/hour per installation

Response headers on every request:
  X-RateLimit-Limit:     5000
  X-RateLimit-Remaining: 4987
  X-RateLimit-Reset:     1712345678   (Unix timestamp)
  X-RateLimit-Used:      13

On 429:
  Retry-After: 1800
  X-RateLimit-Remaining: 0
```

Separate rate limit buckets per dimension: per-user, per-IP, per-IP+unauthenticated. CI pipelines get a higher limit to support automated workflows. Log rate limit hits per API key to proactively notify customers approaching limits.
