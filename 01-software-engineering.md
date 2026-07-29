# 01 · Software Engineering Fundamentals

The AI-specific layers of these roles sit on top of ordinary engineering. Interviewers assume you can write correct, testable integration code before they care whether you know what a reranker is — and a shaky answer here undermines everything you say later about RAG or agents. This file skews toward the code you'll actually write gluing systems together (APIs, retries, tests, SQL) over algorithmic puzzles, because that's what these roles are.

---

## Multiple choice

### Q1.1 · Async fundamentals · [Recall]

In Python, what does `await` actually do when it hits an `async def` function call?

- **A.** Blocks the entire process until the coroutine completes
- **B.** Suspends the current coroutine and returns control to the event loop until the awaited coroutine is ready to resume
- **C.** Spawns the coroutine on a new OS thread and waits for it to join
- **D.** Immediately schedules the coroutine and continues executing the next line without waiting

<details>
<summary>Answer</summary>

**B**

**Why B:** `await` yields control back to the event loop, which can run other coroutines while the awaited one is waiting on I/O. That's the entire point of async — one thread, many in-flight operations.

**Why not A:** That's what a synchronous blocking call does, not `await`. If `await` blocked the process, async would offer nothing over synchronous code.

**Why not C:** Python's `asyncio` is single-threaded by default (barring explicit executor use for CPU-bound work). There's no thread spawned here.

**Why not D:** That describes "fire and forget" (e.g. `asyncio.create_task` without awaiting it), not `await` itself — `await` does wait for the result before the line after it executes.

**Interviewer's likely follow-up:** *"So if async is single-threaded, how does it help with a CPU-bound task like resizing 1,000 images?"* (Answer: it doesn't — async helps with I/O-bound concurrency; CPU-bound work needs multiprocessing or a thread pool executor.)
</details>

### Q1.2 · Concurrency bug · [Applied]

You have a FastAPI endpoint that calls three downstream APIs and combines their results. Written like this:

```python
result_a = await client.get("/a")
result_b = await client.get("/b")
result_c = await client.get("/c")
```

Each call takes ~400ms and they don't depend on each other. What's the most direct fix to cut latency?

- **A.** Switch to synchronous `requests` calls, which are faster for short requests
- **B.** Wrap the three calls in `asyncio.gather()` so they run concurrently
- **C.** Move the calls into three separate background threads using `threading.Thread`
- **D.** Increase the connection pool size on the HTTP client

<details>
<summary>Answer</summary>

**B**

**Why B:** The three calls are independent, so there's no reason to await them sequentially. `asyncio.gather(client.get("/a"), client.get("/b"), client.get("/c"))` starts all three and waits for all to finish — total latency drops from ~1200ms to ~400ms (plus overhead).

**Why not A:** Synchronous requests would be strictly worse — you'd lose the ability to overlap I/O at all, guaranteeing ~1200ms sequential regardless of framework.

**Why not C:** You'd be reintroducing OS thread overhead to solve a problem `asyncio.gather` solves for free within the existing event loop. It would work, but it's the wrong tool and adds GIL contention and thread-safety concerns for no benefit.

**Why not D:** Connection pool size matters if you're bottlenecked on connection reuse across many concurrent requests, but it doesn't address the root issue here: the three calls are being run sequentially by the code, not concurrently limited by the pool.

**Interviewer's likely follow-up:** *"What happens if `/b` raises an exception — what does `gather` do to the other two calls?"* (Answer: by default, `gather` cancels the other pending awaitables and immediately re-raises the first exception, unless you pass `return_exceptions=True`, in which case exceptions are returned as results alongside successes.)
</details>

### Q1.3 · REST idempotency · [Applied]

A mobile client sends a `POST /payments` request to charge a card. The client gets a network timeout, doesn't know if the server processed it, and retries the exact same request. What's the correct way to prevent a duplicate charge?

- **A.** Have the client wait longer before timing out
- **B.** Require an `Idempotency-Key` header on the request that the server uses to deduplicate retried requests
- **C.** Make the endpoint `GET` instead of `POST` so retries are automatically safe
- **D.** Add a confirmation step where the user manually re-approves before any retry

<details>
<summary>Answer</summary>

**B**

**Why B:** This is the standard pattern (used by Stripe, among others): the client generates a unique key per logical operation and sends it every time, including on retries. The server stores the key with the result of the first execution and returns that cached result for any repeat, instead of re-executing the charge.

**Why not A:** A longer timeout reduces how often this scenario occurs but doesn't fix it — the client still can't distinguish "still processing" from "failed," and the underlying request still isn't idempotent.

**Why not C:** `POST` is semantically correct here — you're creating a resource (a charge) with side effects. Changing the verb doesn't change the fact that the operation isn't naturally idempotent; you'd just be lying about the semantics.

**Why not D:** This punts the problem to the user and adds friction to every request, not just the failure case. It's a workaround, not a fix, and it doesn't scale to server-to-server integrations with no human in the loop.

**Interviewer's likely follow-up:** *"Where do you store the idempotency key and the result, and how long do you keep it?"* (Answer: typically a database table or fast key-value store keyed by the idempotency key, storing status and result; TTL is usually 24 hours, long enough to cover realistic retry windows but bounded so storage doesn't grow unbounded.)
</details>

### Q1.4 · API versioning · [Recall]

Which of the following is generally considered the most maintainable way to version a public REST API?

- **A.** Embed the version in the URL path, e.g. `/v1/users`
- **B.** Never version — just change the response shape in place and let clients adapt
- **C.** Use a query parameter for every field that might change, e.g. `?legacy_format=true`
- **D.** Maintain one branch per client and deploy each separately

<details>
<summary>Answer</summary>

**A**

**Why A:** URL-path versioning (`/v1/`, `/v2/`) is explicit, cacheable, easy to route at the infrastructure layer, and immediately visible in logs and documentation. It's not the only valid approach (header-based versioning is a legitimate alternative), but it's the most common and most maintainable default for a public API with external clients you don't control the deploy schedule of.

**Why not B:** Breaking changes in place will silently break every client on their own schedule, with no way for them to opt in or out. This is the single most common cause of "why did production break with no code change on our side" incidents in API consumers.

**Why not C:** Flag-per-field creates a combinatorial explosion of supported states and makes it nearly impossible to reason about what any given request will actually return. It also doesn't solve the core problem — you still need a plan for deprecating the old behavior.

**Why not D:** Branch-per-client isn't versioning at all — it's forking your codebase per customer, which is a maintenance nightmare that grows linearly with your customer count.

**Interviewer's likely follow-up:** *"How do you decide when it's safe to sunset v1?"* (Answer: usage metrics on the old version, an explicit deprecation window communicated to clients, ideally with a sunset header/date, and confirmation that no active client dependency remains before removal.)
</details>

### Q1.5 · Pagination · [Applied]

A `/tickets` endpoint uses offset-based pagination (`?offset=1000&limit=50`). Under load, users report seeing duplicate and missing tickets when paging through results while new tickets are being created concurrently. What's happening?

- **A.** The database index is corrupted
- **B.** Offset pagination is sensitive to inserts/deletes shifting the row order between page requests
- **C.** The `limit` value is too high
- **D.** The client is caching stale pages

<details>
<summary>Answer</summary>

**B**

**Why B:** Offset pagination re-runs `ORDER BY ... OFFSET N LIMIT M` on every page request. If a row is inserted or deleted between two page fetches, every row's logical position shifts — rows can be skipped (appearing "missing") or repeated (appearing "duplicate"), even though nothing is actually wrong with the data.

**Why not A:** Index corruption would cause missing or incorrect rows consistently, not a pattern specifically correlated with concurrent writes during pagination — and it's a rare failure mode compared to the well-known offset pagination issue.

**Why not C:** `limit` size affects how many rows come back per page, not whether the underlying rows shift between requests. A smaller limit would actually make the problem worse (more page boundaries = more chances to catch a shift), not better.

**Why not D:** Client-side caching could cause stale data, but it wouldn't specifically cause the missing/duplicate pattern tied to concurrent inserts — it would just show old data.

**Interviewer's likely follow-up:** *"What would you switch to instead?"* (Answer: cursor/keyset pagination — paginate on a stable, unique, ordered column like `id` or `created_at + id`, e.g. `WHERE id > last_seen_id ORDER BY id LIMIT 50`, which is stable under concurrent inserts because it doesn't depend on row position.)
</details>

### Q1.6 · Retry semantics · [Applied]

A background worker calls a third-party API that occasionally returns `503 Service Unavailable`. The current code retries immediately, up to 5 times, with no delay. Under moderate load from that third party, this makes things measurably worse. Why?

- **A.** Immediate retries increase load on the already-struggling downstream service, worsening the outage — this is a retry storm
- **B.** `503` should never be retried, only `500`
- **C.** The worker's timeout is too short
- **D.** 5 retries is too few

<details>
<summary>Answer</summary>

**A**

**Why A:** `503` typically means "I'm overloaded, back off." Immediately retrying five times per failed request, multiplied across every worker instance hitting the same downstream, adds load exactly when the service can least handle it — turning a transient blip into a sustained outage. This is the textbook retry storm / thundering herd pattern.

**Why not B:** `503` is actually one of the more retry-appropriate status codes (it's explicitly meant to signal "try again later"), as long as you back off. The problem here isn't retrying `503` — it's retrying it with no delay.

**Why not C:** Timeout length is unrelated to this specific symptom; the issue described is about retry cadence, not about requests timing out.

**Why not D:** Retry count isn't the root cause — even 2 immediate retries across many concurrent workers can pile on load. Fixing the count without fixing the spacing wouldn't resolve the underlying problem.

**Interviewer's likely follow-up:** *"How would you fix it?"* (Answer: exponential backoff with jitter — see E1.2 — and ideally respect a `Retry-After` header if the API provides one, plus a circuit breaker to stop retrying entirely once failure rate crosses a threshold.)
</details>

### Q1.7 · Backoff and jitter · [Design]

You're designing the retry policy for a client SDK that many services will use to call your API. Why add random jitter to exponential backoff instead of using pure exponential delays (1s, 2s, 4s, 8s...)?

- **A.** Jitter makes each individual retry faster
- **B.** Pure exponential backoff causes many clients that failed at the same time to retry in synchronized waves, hammering the server in bursts; jitter spreads those retries out
- **C.** Jitter is required by the HTTP spec for any client that retries
- **D.** Jitter reduces the total number of retries needed

<details>
<summary>Answer</summary>

**B**

**Why B:** If a server hiccup causes 10,000 clients to fail simultaneously, pure exponential backoff means they all retry at exactly 1s, then all at 3s, then all at 7s (cumulative) — synchronized waves that can each look like a new spike of load. Adding randomness (e.g. "full jitter": `random(0, base * 2^attempt)`) desynchronizes them so retries arrive spread out over time instead of in a burst.

**Why not A:** Jitter typically makes an individual retry's timing less predictable, not faster — it can make any single client's average wait slightly longer, in exchange for smoothing the aggregate load pattern across all clients.

**Why not C:** There's no HTTP spec requirement for jitter. It's a widely adopted best practice (AWS has written specifically about this), not a protocol rule.

**Why not D:** Jitter doesn't change how many retries a given client makes — it changes when those retries happen relative to other clients' retries.

**Interviewer's likely follow-up:** *"What's the difference between 'full jitter' and 'equal jitter,' and when might you prefer one?"* (Answer: full jitter picks a delay uniformly from `[0, cap]`, maximizing spread but allowing very short waits; equal jitter keeps half the exponential delay fixed and randomizes the other half, giving more predictable minimum spacing at the cost of slightly less spread — full jitter is usually preferred for maximizing throughput under contention.)
</details>

### Q1.8 · Error handling · [Recall]

What's the main problem with this pattern, seen in a lot of integration code?

```python
try:
    result = external_api.call()
except Exception:
    result = None
```

- **A.** It's too verbose
- **B.** It silently swallows every possible failure — including bugs, auth errors, and malformed responses — indistinguishably from an expected "no result" case, making debugging and monitoring nearly impossible
- **C.** `Exception` should be `BaseException`
- **D.** `result = None` should be `result = {}`

<details>
<summary>Answer</summary>

**B**

**Why B:** A bare `except Exception` catches everything from a network timeout to a `KeyError` from a code bug to a 401 from expired credentials — and treats them all identically, as "no result." This makes downstream code silently work with missing data, hides real bugs, and gives you no signal in monitoring that anything went wrong. Catching specific exceptions (or at minimum, logging and re-raising) preserves the information needed to actually operate the system.

**Why not A:** Verbosity isn't the issue — a properly narrow `except` block with logging would be just as short and vastly more useful.

**Why not C:** `BaseException` is broader still (it includes `SystemExit`, `KeyboardInterrupt`) — catching it would be even worse, not a fix.

**Why not D:** The choice of sentinel value (`None` vs `{}`) is a minor detail compared to the fact that all error types are conflated. Changing the sentinel doesn't address the actual problem.

**Interviewer's likely follow-up:** *"How would you rewrite this for production?"* (Answer: catch specific exceptions you expect and can handle meaningfully, e.g. `requests.Timeout` or a specific API error class; log unexpected exceptions with context before deciding whether to re-raise, return a typed error, or degrade gracefully — but never silently discard the exception type and traceback.)
</details>

### Q1.9 · TypeScript idioms · [Applied]

A teammate writes this to type an API response:

```typescript
function parseUser(data: any): User {
  return data as User;
}
```

What's the issue?

- **A.** `as User` is invalid TypeScript syntax
- **B.** This is a type assertion, not validation — if `data` doesn't actually match `User`'s shape at runtime, TypeScript won't catch it, and you get a silent type lie that surfaces as a confusing runtime error later
- **C.** `any` should be `unknown` for performance reasons
- **D.** Functions can't return interfaces in TypeScript

<details>
<summary>Answer</summary>

**B**

**Why B:** `as User` tells the compiler "trust me, this is a `User`" — it performs no runtime check. If the API response is missing a field or has the wrong shape (which happens constantly with real APIs — versions drift, fields get renamed), the code compiles fine and then blows up (or worse, silently misbehaves) at runtime, at a point far from where the bad data actually entered the system.

**Why not A:** The syntax is valid — that's exactly the problem being tested here; it *looks* safe because it type-checks.

**Why not C:** `any` vs `unknown` is a real distinction (`unknown` forces you to narrow before use, which is safer), but it's not the core issue in this snippet — the core issue is the unchecked assertion, and switching `any` to `unknown` as the parameter type wouldn't be about performance, it'd be about safety, and even then wouldn't fix the `as User` cast itself.

**Why not D:** Functions returning interface types is completely normal TypeScript and not the problem here.

**Interviewer's likely follow-up:** *"How would you fix it?"* (Answer: runtime validation with a schema library like Zod — `UserSchema.parse(data)` — which throws a descriptive error immediately if the shape doesn't match, at the boundary where untrusted data enters the system, instead of trusting a compile-time-only assertion.)
</details>

### Q1.10 · Testing non-determinism · [Design]

You're writing tests for a feature that calls an LLM to summarize text, and the summary wording varies slightly between runs even at temperature 0 (due to floating-point non-associativity across GPU batch sizes, among other things). How should you test this?

- **A.** Assert exact string equality against a recorded "golden" output
- **B.** Skip testing this code path entirely since LLM output isn't deterministic
- **C.** Assert on structural/semantic properties that must hold regardless of exact wording — e.g. output length bounds, presence of key entities, valid JSON shape, or use an LLM-as-judge check for semantic similarity to a reference — rather than exact string matching
- **D.** Mock the LLM call to always return a fixed string, and don't test the real integration at all

<details>
<summary>Answer</summary>

**C**

**Why C:** Since exact output isn't stable, the test needs to check properties that *are* stable: does the summary contain the required entities, is it under the length limit, does it parse as valid JSON if that's the contract, does a semantic-similarity or LLM-judge check score it above a threshold against a reference. This tests what actually matters (does the feature work) without being brittle to harmless wording drift.

**Why not A:** Exact string equality will fail intermittently for reasons that have nothing to do with a real bug, making the test noisy and eventually ignored — the worst outcome for a test suite.

**Why not B:** Not testing at all leaves a real, user-facing code path uncovered. Non-determinism is a property to design tests around, not an excuse to skip testing.

**Why not D:** Mocking is valuable for *unit* tests that check your code's logic around the LLM call (prompt construction, error handling, retry behavior) in isolation — but if you *never* test against the real API, you won't catch integration issues like schema drift, auth problems, or actual quality regressions. You need both: mocked unit tests for logic, and a smaller set of real/semi-real integration tests with property-based assertions.

**Interviewer's likely follow-up:** *"How would you keep a real-API integration test from being flaky in CI?"* (Answer: run it less frequently than the full unit suite — e.g. nightly or pre-merge only — use a loose enough threshold on the semantic check to absorb normal variance, retry once on transient API errors before failing, and keep it separate from the fast unit suite so an LLM hiccup doesn't block every PR.)
</details>

### Q1.11 · Mocking external APIs · [Applied]

Your integration test mocks a payment provider's SDK by patching `PaymentClient.charge()` to always return a success object. Six months later, the provider changes their response schema, but this test — and all tests like it — keep passing while production breaks. What's the underlying testing gap?

- **A.** The mock should have been written in a different testing framework
- **B.** There was no contract test verifying the mock's shape still matches the real API's actual current response shape
- **C.** The test should have used `time.sleep()` before asserting
- **D.** Mocking is inherently unreliable and should never be used

<details>
<summary>Answer</summary>

**B**

**Why B:** Mocks encode an assumption about what the real dependency returns, frozen at the moment you wrote the mock. Nothing keeps that assumption in sync with reality unless something else checks it — a contract test (hitting the real sandbox API periodically, or using a shared schema/fixture validated against the provider's actual docs or OpenAPI spec) is exactly the mechanism that would have caught this drift before it hit production.

**Why not A:** This is a testing-strategy gap, not a framework limitation — every mocking framework in every language has this same blind spot by design.

**Why not C:** Timing has nothing to do with a schema mismatch; adding a sleep wouldn't surface a stale mock.

**Why not D:** Mocking is essential for fast, deterministic unit tests — the fix isn't to abandon mocking, it's to pair mocked unit tests with periodic contract/integration tests that validate the mock's assumptions against reality.

**Interviewer's likely follow-up:** *"Where does responsibility for catching this sit — engineering process, or something you can automate?"* (Answer: ideally automated — a scheduled CI job that hits the provider's sandbox and validates the response against the same schema the mock uses, so drift is caught within a day rather than at the next manual QA pass.)
</details>

### Q1.12 · Unit vs integration · [Recall]

Which statement best distinguishes a unit test from an integration test?

- **A.** Unit tests are always faster; integration tests are always slower — that's the only real difference
- **B.** A unit test verifies a single component in isolation (dependencies mocked/stubbed); an integration test verifies that multiple real components work correctly together
- **C.** Unit tests use `assert`; integration tests use a testing framework
- **D.** Integration tests are written by QA; unit tests are written by developers

<details>
<summary>Answer</summary>

**B**

**Why B:** This is the core distinction — scope of what's under test and what's real vs faked. Speed is a common *consequence* of that distinction, not the definition itself.

**Why not A:** Speed is usually correlated but it's a side effect, not the defining property — a unit test with a huge in-memory dataset can be slower than a trivial integration test against a fast local service.

**Why not C:** Both typically use the same testing framework and assertion style; this isn't a distinguishing factor at all.

**Why not D:** In modern engineering practice both are typically written by the developer who wrote the feature — this isn't a role-based distinction.

**Interviewer's likely follow-up:** *"Where do you draw the line for code that calls an LLM API — is a test that calls the real model a unit test or an integration test?"* (Answer: integration test — it depends on a real external component whose behavior you don't fully control; the unit-test equivalent would mock the LLM client and test your surrounding logic.)
</details>

### Q1.13 · Git workflow · [Applied]

You're three commits into a feature branch when you realize commit 2 introduced a bug that's now also present in commit 3's changes. What's the cleanest way to fix commit 2 without losing the legitimate changes in commit 3?

- **A.** `git reset --hard` to before commit 2 and redo all the work
- **B.** `git rebase -i` to edit commit 2 directly, then let Git replay commit 3 on top with the fix included
- **C.** Add a new commit 4 that reverts commit 2 entirely
- **D.** Force-push over the branch with a fresh single commit containing everything

<details>
<summary>Answer</summary>

**B**

**Why B:** Interactive rebase lets you stop at commit 2, amend it with the fix, and Git automatically replays commit 3 on top of the corrected commit 2 — preserving a clean, logical history where each commit is individually correct, without losing any work.

**Why not A:** `reset --hard` discards commit 3's work entirely — you'd have to manually redo it, which is unnecessary risk and effort when rebase does this safely.

**Why not C:** Reverting commit 2 with a new commit removes its changes going forward, but commit 3 still depends on the buggy version existing in history — you'd likely need a second revert or manual fix anyway, and you'd end up with a messier history (bug, then revert, then fix) instead of a clean one.

**Why not D:** This works but destroys the granular history and makes code review of what changed and why much harder — it's a blunt instrument for a problem `rebase -i` solves precisely.

**Interviewer's likely follow-up:** *"This branch has already been pushed and a teammate pulled it. Does that change your approach?"* (Answer: yes — rebasing rewrites history, so anyone who already pulled the branch will have a diverged copy; you'd need to coordinate the force-push and have them reset to the new history, or avoid rebasing already-shared branches in favor of the revert-and-fix approach instead.)
</details>

### Q1.14 · SQL basics · [Applied]

A query joining `orders` to `customers` is timing out on a table with 50 million orders. `EXPLAIN` shows a sequential scan on `orders.customer_id`. What's the most likely fix?

- **A.** Rewrite the join as a subquery
- **B.** Add an index on `orders.customer_id`
- **C.** Increase the database's memory allocation
- **D.** Switch from `INNER JOIN` to `LEFT JOIN`

<details>
<summary>Answer</summary>

**B**

**Why B:** A sequential scan on the join column on a 50-million-row table means the database is reading every row to find matches instead of using an index to jump directly to relevant rows. Adding an index on `orders.customer_id` (the foreign key being joined on) lets the planner use an index scan or index-nested-loop join instead, which is dramatically faster at this scale.

**Why not A:** Restructuring as a subquery doesn't change the fundamental access pattern — without an index, the subquery would still require scanning the same rows.

**Why not C:** More memory can help with sorting/hashing large result sets, but it doesn't fix the root cause here, which is the missing index causing a full scan; the query would still be slow, just slow with more RAM available.

**Why not D:** Changing join type changes which rows are included in the result (`LEFT JOIN` keeps unmatched `orders` rows), it doesn't address performance — and it would actually change the query's semantics, likely incorrectly, purely as a performance hack.

**Interviewer's likely follow-up:** *"What's the tradeoff of adding that index?"* (Answer: indexes speed up reads but slow down writes — every `INSERT`/`UPDATE`/`DELETE` on `orders` now also has to update the index — and they consume additional storage, so you add indexes deliberately based on actual query patterns, not preemptively on every column.)
</details>

### Q1.15 · Complexity · [Recall]

You're iterating over a list of 10,000 user IDs and, for each one, checking membership in a Python `list` of 5,000 "banned" IDs using `in`. What's the time complexity of the overall check, and what's the simple fix?

- **A.** O(n), already optimal
- **B.** O(n × m) because `in` on a list is O(m); converting the banned list to a `set` makes membership checks O(1) average, dropping the overall check to O(n)
- **C.** O(n log n), fix by sorting first
- **D.** O(1), Python lists have constant-time membership checks

<details>
<summary>Answer</summary>

**B**

**Why B:** `x in list` scans the list linearly — O(m) per check, done n times, gives O(n × m) = 50,000,000 comparisons in the worst case here. Converting the banned list to a `set` (hash table) makes each membership check O(1) on average, so the total drops to O(n) — a difference that's very noticeable at this scale (10,000 × 5,000 vs 10,000 checks).

**Why not A:** O(n) would be true only if the membership check itself were O(1), which it isn't for a list — this option describes the *target* state, not the current one.

**Why not C:** Sorting plus binary search would get you to O(n log m), which is better than the naive approach but still strictly worse than the O(n) you get almost for free by using a `set` — and it's more code for a worse result.

**Why not D:** Python lists do not have constant-time membership checks — that's a common misconception carried over from thinking about hash-based structures; only `set` and `dict` give average O(1) membership.

**Interviewer's likely follow-up:** *"When would a set NOT be the right structure for this, even if it's faster?"* (Answer: if you need the banned list's order preserved, or need to store non-hashable items, or the list is tiny and readability of a simple list outweighs a negligible performance difference — premature optimization on a 10-item list isn't worth the marginal complexity.)
</details>

### Q1.16 · Idempotency vs safety · [Design]

Which HTTP method pairing correctly distinguishes "safe" (no side effects) from "idempotent" (repeatable with the same effect)?

- **A.** `GET` is safe and idempotent; `PUT` is idempotent but not safe; `POST` is neither by default
- **B.** All HTTP methods are safe by definition
- **C.** `POST` is always idempotent because it returns a 201
- **D.** `DELETE` is safe because it doesn't return data

<details>
<summary>Answer</summary>

**A**

**Why A:** `GET` should have no side effects (safe) and calling it repeatedly returns the same result without changing state further (idempotent). `PUT` (full replace) has a side effect but calling it N times with the same payload leaves the resource in the same end state as calling it once (idempotent, not safe). `POST` typically creates a new resource each time by default — calling it twice creates two resources, so it's neither safe nor idempotent unless you explicitly add idempotency handling (see Q1.3).

**Why not B:** This directly contradicts the definition of "safe" — `POST` and `DELETE` explicitly have side effects and are not safe methods.

**Why not C:** Returning `201` is just a status code convention for "created" — it has no bearing on whether repeated calls produce the same end state. Calling `POST /orders` twice normally creates two orders, which is the opposite of idempotent.

**Why not D:** "Safe" specifically means no side effects on server state — `DELETE` removes a resource, which is very much a side effect, regardless of whether it returns a response body. `DELETE` is idempotent (deleting an already-deleted resource leaves the same end state — gone) but it is not safe.

**Interviewer's likely follow-up:** *"Is `PATCH` idempotent?"* (Answer: not by default/spec — it depends on the semantics of the patch operation; a `PATCH` that sets a field to an absolute value is idempotent, but a `PATCH` like "increment counter by 1" is not, since repeating it changes the result each time.)
</details>

### Q1.17 · Async pitfalls · [Applied]

A developer writes this to process 1,000 API calls "concurrently":

```python
tasks = [asyncio.create_task(fetch(url)) for url in urls]
results = await asyncio.gather(*tasks)
```

The target API starts returning `429 Too Many Requests` almost immediately. What's missing?

- **A.** `asyncio.gather` doesn't actually run tasks concurrently
- **B.** There's no concurrency limit — all 1,000 requests fire near-simultaneously; a semaphore (e.g. `asyncio.Semaphore(20)`) should bound how many run at once
- **C.** `create_task` should be replaced with `await fetch(url)` in the list comprehension
- **D.** The URLs need to be sorted first

<details>
<summary>Answer</summary>

**B**

**Why B:** This code is *correct* in that it does run concurrently — that's exactly the problem. Firing 1,000 requests at once will trip almost any API's rate limiting. The fix is to bound concurrency with something like a semaphore, so only N requests are in flight at a time, respecting the target API's actual rate limits.

**Why not A:** `gather` does run tasks concurrently — that's its entire purpose and it's working exactly as designed here; the issue is the *lack of a limit* on that concurrency, not a failure of `gather` to be concurrent.

**Why not C:** Doing that would make the calls sequential instead of concurrent, which fixes the rate-limit issue but by throwing away all the benefit of async — a working but needlessly slow fix compared to bounded concurrency.

**Why not D:** Sorting the URLs has no relationship to request rate or concurrency — the API doesn't care what order requests arrive in, only how many arrive per unit time.

**Interviewer's likely follow-up:** *"Show me roughly how you'd wire a semaphore into this."* (Answer: wrap each fetch in `async with semaphore: return await fetch(url)`, where `semaphore = asyncio.Semaphore(20)` is created once outside the loop — this caps in-flight requests at 20 regardless of how many tasks are created.)
</details>

### Q1.18 · Error handling in APIs · [Design]

You're designing error responses for an internal API other teams will integrate against. Which approach gives consumers the most actionable information?

- **A.** Return HTTP 500 with an empty body for all failures, to avoid leaking internals
- **B.** Return the appropriate HTTP status code plus a structured JSON body with a machine-readable error code, a human-readable message, and (where applicable) which field caused the problem
- **C.** Return HTTP 200 always, with an `"error": true` field in the body so clients don't need special status-code handling
- **D.** Return a plain-text stack trace so engineers can debug directly from the response

<details>
<summary>Answer</summary>

**B**

**Why B:** This gives consumers everything they need to build correct handling: the status code lets infrastructure (load balancers, monitoring, generic HTTP clients) react correctly without parsing the body; the machine-readable code lets client code branch reliably (`error_code == "INSUFFICIENT_FUNDS"`) without string-matching a message that might change wording; the human message helps engineers debug; the field reference speeds up fixing validation errors specifically.

**Why not A:** Returning 500 for everything (including client errors like bad input) makes it impossible for consumers to distinguish "you sent something wrong, fix your request" from "we broke, retry later or page us" — this collapses a critical distinction that determines what the caller should even do next.

**Why not C:** This breaks every piece of standard HTTP tooling (caching, monitoring, load balancer health checks, generic retry logic) that relies on status codes, forcing every consumer to parse the body just to know if a call succeeded — a significant, unnecessary integration burden.

**Why not D:** Stack traces leak internal implementation details (file paths, library versions, sometimes secrets in variable state) to API consumers, which is both a security risk and unhelpful to external integrators who can't act on your internal trace anyway — that information belongs in server-side logs, correlated by a request ID.

**Interviewer's likely follow-up:** *"How would a client correlate a specific failed request with your server-side logs for deeper debugging?"* (Answer: include a request ID / trace ID in both the error response and the server logs, so the consumer can hand you that ID for immediate lookup instead of guessing at timestamps.)
</details>

### Q1.19 · Testing philosophy · [Recall]

Which is NOT a good reason to write a test?

- **A.** To lock in the correct behavior of a bug fix so it doesn't regress
- **B.** To document expected behavior for future maintainers
- **C.** To hit 100% code coverage as reported by a coverage tool, regardless of what the test actually verifies
- **D.** To give you confidence to refactor without breaking existing behavior

<details>
<summary>Answer</summary>

**C**

**Why C is the exception being asked for:** Coverage percentage is a proxy metric, not a goal in itself. A test that executes a line without meaningfully asserting on its behavior (e.g. calling a function and asserting nothing, or asserting only that it "didn't throw") inflates coverage while providing none of the real value tests are supposed to provide — regression protection, documentation, or refactor confidence. Chasing the number produces exactly this kind of hollow test.

**Why A, B, D are legitimate:** These are the three real reasons to write tests — regression prevention, living documentation of intended behavior, and enabling safe change over time. All are genuinely valuable and none has the "gaming a metric" problem that C has.

**Interviewer's likely follow-up:** *"How would you tell, in code review, if a test was written to hit coverage rather than to actually verify behavior?"* (Answer: check whether the assertions actually constrain the output in a meaningful way — a test with weak or missing assertions, or one that would still pass if you deliberately broke the function's logic, is a coverage-only test.)
</details>

### Q1.20 · SQL basics · [Recall]

What's the key difference between `WHERE` and `HAVING` in a SQL query with `GROUP BY`?

- **A.** They're interchangeable — both filter rows
- **B.** `WHERE` filters rows before grouping/aggregation happens; `HAVING` filters groups after aggregation, and is required if you want to filter on an aggregate value like `COUNT(*)` or `SUM(x)`
- **C.** `HAVING` is deprecated in favor of `WHERE`
- **D.** `WHERE` only works with `SELECT *`

<details>
<summary>Answer</summary>

**B**

**Why B:** `WHERE` operates on individual rows before they're grouped, so it can't reference an aggregate function's result (`COUNT`, `SUM`, `AVG`, etc.) because those don't exist yet at that stage. `HAVING` runs after `GROUP BY` has produced aggregated groups, so it can filter on those aggregate values — e.g. `SELECT customer_id, COUNT(*) FROM orders GROUP BY customer_id HAVING COUNT(*) > 5`.

**Why not A:** They operate at different stages of query execution and aren't interchangeable — trying to write `WHERE COUNT(*) > 5` will error in most databases, precisely because `WHERE` runs before aggregation exists.

**Why not C:** `HAVING` is standard, actively used SQL, not deprecated — it does something `WHERE` structurally cannot.

**Why not D:** `WHERE` works with any column selection, not just `SELECT *` — this option doesn't reflect any real constraint in SQL.

**Interviewer's likely follow-up:** *"Can you use both `WHERE` and `HAVING` in the same query, and if so, in what order do they logically apply?"* (Answer: yes — `WHERE` filters individual rows first, then `GROUP BY` aggregates the remaining rows, then `HAVING` filters the resulting groups; using both lets you cheaply exclude irrelevant rows before the more expensive aggregation step.)
</details>

### Q1.21 · Retry semantics · [Design]

Which category of error should generally NOT be retried automatically by client code?

- **A.** Network timeouts
- **B.** `503 Service Unavailable`
- **C.** `400 Bad Request` (malformed request body)
- **D.** `429 Too Many Requests`

<details>
<summary>Answer</summary>

**C**

**Why C is the one NOT to retry:** A `400` means the request itself is malformed — retrying the exact same request will produce the exact same `400` every time, since nothing about the server state changed; it's a client-side bug, not a transient condition. Retrying wastes a call and delays the client from surfacing the real problem (fix the request).

**Why A, B, D are appropriate to retry:** Timeouts and `503`s are classically transient — the server or network may recover between attempts. `429` is explicitly a "you're going too fast, try again" signal, and should be retried with backoff (ideally honoring any `Retry-After` header) rather than treated as a hard failure.

**Interviewer's likely follow-up:** *"What about a 401 Unauthorized — retry or not?"* (Answer: not with the same credentials — retrying an expired/invalid token will fail identically every time; the correct action is to refresh the token once and retry the request exactly once with the new token, not to blindly retry the original failing request.)
</details>

### Q1.22 · Testing mocking boundaries · [Design]

You're writing tests for a service that calls an internal microservice you also own. Should you mock that internal call in your unit tests?

- **A.** No — always call the real internal service in every test to be safe
- **B.** Yes for unit tests of your service's own logic (mock the boundary so tests are fast and isolated); but also maintain a smaller set of integration/contract tests that exercise the real call, since you own both sides and schema drift between them is entirely possible
- **C.** No, mocking is only appropriate for third-party APIs you don't control
- **D.** It doesn't matter since it's an internal service you control

<details>
<summary>Answer</summary>

**B**

**Why B:** Owning both sides doesn't eliminate the risk of drift — someone on the other team can change that service's response shape without you noticing, exactly like Q1.11's third-party example. Mocking gives you fast, deterministic unit tests of *your* logic; a smaller integration/contract test suite protects against the assumption in that mock going stale. This is the same reasoning as testing against an external API, just with an internal dependency.

**Why not A:** Calling the real service in every test makes your test suite slow, flaky (dependent on that service's uptime and state), and couples unrelated services' test runs together — a bad tradeoff for the majority of tests that are really about your own logic.

**Why not C:** The "own vs third-party" distinction doesn't change the fundamental testing tradeoff — both benefit from the same mock-for-speed, contract-test-for-safety pattern. Ownership makes coordination easier, not the testing strategy different.

**Why not D:** Ownership reduces communication friction (you can just ask the other team, or read their code) but doesn't prevent drift — code changes independently on both sides all the time, especially across team boundaries within the same company.

**Interviewer's likely follow-up:** *"If you owned both services, would you rather share a schema definition between them?"* (Answer: yes, ideally — a shared schema/type definition (protobuf, a shared OpenAPI spec, or a shared package) turns "drift" into a compile-time or CI-time failure instead of a runtime surprise.)
</details>

### Q1.23 · Concurrency vs parallelism · [Recall]

What's the distinction between concurrency and parallelism?

- **A.** They're synonyms
- **B.** Concurrency is about structuring a program to handle multiple tasks that can be in progress at once (possibly interleaved on one core); parallelism is about actually executing multiple tasks at the exact same instant, which requires multiple cores/threads
- **C.** Concurrency requires multiple CPUs; parallelism doesn't
- **D.** Parallelism only applies to distributed systems

<details>
<summary>Answer</summary>

**B**

**Why B:** A single-threaded `asyncio` event loop is concurrent (it manages many in-flight I/O operations, switching between them) but not parallel (only one line of Python bytecode executes at any given instant). True parallelism needs multiple cores actually doing work simultaneously, e.g. via multiprocessing.

**Why not A:** Conflating them is a very common but incorrect simplification — the distinction specifically matters for choosing `asyncio` (good for I/O-bound concurrency) vs `multiprocessing` (needed for CPU-bound parallelism).

**Why not C:** It's backwards — concurrency is achievable on a single core (that's the whole point of async/event loops); parallelism is what actually requires multiple cores.

**Why not D:** Parallelism applies just as much within a single multi-core machine (e.g. `multiprocessing.Pool`) as across a distributed system — it's not scoped to distributed systems specifically.

**Interviewer's likely follow-up:** *"You have a CPU-heavy image-processing step in an otherwise I/O-bound async pipeline. What do you do?"* (Answer: offload it to a process pool executor, e.g. `loop.run_in_executor(process_pool, cpu_heavy_fn, arg)`, so the CPU-bound work runs in a separate process and doesn't block the event loop that's handling everything else.)
</details>

### Q1.24 · API design · [Applied]

Daily Sparks Events' booking API needs an endpoint for a client's staff to check event availability for a date range. Which design better fits REST conventions?

- **A.** `POST /checkAvailability` with a JSON body containing the date range
- **B.** `GET /events/availability?start=2026-08-01&end=2026-08-07`
- **C.** `POST /events/availability/check/run`
- **D.** `GET /events` with the date range embedded in a custom header

<details>
<summary>Answer</summary>

**B**

**Why B:** This is a read-only, idempotent, cacheable operation with no side effects — the textbook case for `GET`. Query parameters are the standard place for filter criteria like a date range, and the URL stays self-descriptive, cacheable by intermediaries, and bookmarkable/shareable, all of which are useful properties for an availability check.

**Why not A:** Using `POST` for a pure read implies a side effect that doesn't exist here, breaks HTTP caching semantics, and goes against the convention that `GET` is for retrieval — reserve `POST` for operations that create or mutate state.

**Why not C:** This mimics RPC-style naming (`.../run`) rather than REST resource semantics, and stacks unnecessary path segments for what should be a simple resource query — it also doesn't fix the underlying `POST`-for-a-read issue if paired with that verb.

**Why not D:** Hiding filter criteria in a custom header instead of the URL makes the request harder to debug, log, cache, and share (you can't just paste a URL to reproduce it), for no real benefit over query parameters.

**Interviewer's likely follow-up:** *"How would you handle a client wanting recurring weekly availability instead of a single range?"* (Answer: either extend the query parameters, e.g. `?recurrence=weekly&occurrences=4`, or if the shape of "recurring availability" is complex enough, treat it as its own resource/sub-resource rather than overloading one endpoint's parameters indefinitely.)
</details>

### Q1.25 · Time complexity · [Design]

You need to deduplicate a list of 2 million log entries (objects, not primitives) by an `id` field, preserving the first occurrence's order. What approach avoids an O(n²) blowup?

- **A.** Nested loop comparing every pair of entries
- **B.** Iterate once, tracking seen `id`s in a `set`; append to the result list only if the `id` hasn't been seen, then add it to the set — O(n) overall
- **C.** Sort the list first by every field, then remove adjacent duplicates
- **D.** Use `list.count()` inside the loop to check for duplicates before appending

<details>
<summary>Answer</summary>

**B**

**Why B:** A single pass with a `set` for O(1) average membership checks gives O(n) total time and preserves insertion order naturally, since you're building the result list in the same order you iterate. This is the standard pattern for this exact problem.

**Why not A:** Comparing every pair is O(n²) — at 2 million entries that's on the order of 4 trillion comparisons, computationally infeasible.

**Why not C:** Sorting is O(n log n), which is better than O(n²) but still strictly worse than the O(n) achievable with a set — and it destroys the "preserve first occurrence's order" requirement unless you do extra work to restore original order afterward, which defeats the point.

**Why not D:** `list.count()` itself scans the entire list each call — calling it inside a loop reintroduces O(n²) behavior, the exact problem being avoided, just hidden inside a built-in method call.

**Interviewer's likely follow-up:** *"What if the entries aren't hashable objects, only dicts with an `id` key?"* (Answer: you don't need the whole object to be hashable — track just the `id` values (which are typically strings/ints, already hashable) in the set, and append the full dict object to the result list when its `id` is new.)
</details>

### Q1.26 · Testing · [Applied]

A pull request adds a new feature and includes tests, but every test mocks so much of the system that the test would still pass even if the actual feature logic were deleted and replaced with `return None`. What's the review feedback?

- **A.** Approve — tests exist, that satisfies the coverage requirement
- **B.** The tests aren't actually verifying the feature's behavior — they're testing that the mocks were called, not that the logic produces correct output; ask for assertions on actual return values/behavior tied to realistic inputs
- **C.** Request that the PR add more tests, doubling the current count
- **D.** Reject because mocking should never be used in tests

<details>
<summary>Answer</summary>

**B**

**Why B:** This is the "coverage theater" failure mode from Q1.19 in concrete PR form — tests that exercise code paths without meaningfully constraining behavior give false confidence. The fix isn't more tests or fewer mocks categorically, it's tests whose assertions would actually fail if the real logic were broken — often by asserting on real return values given realistic input, with only true external boundaries (network calls, the DB) mocked.

**Why not A:** Test *count* or presence is not the bar — a test suite that can't detect the feature being deleted has provided approximately zero regression protection, regardless of how many tests exist.

**Why not C:** Doubling a set of tests that don't actually assert meaningful behavior just doubles the false confidence — quantity doesn't fix a quality problem.

**Why not D:** Mocking itself isn't the problem (it's necessary and correct to isolate units) — the problem is *what's* being asserted, not that mocking was used at all.

**Interviewer's likely follow-up:** *"How do you catch this kind of test in review quickly, without reading every line?"* (Answer: a fast heuristic — read the assertions first, before the setup/mocking code; if you can't tell what correct behavior looks like just from the `assert` lines, the test probably isn't verifying enough.)
</details>

### Q1.27 · Git workflow · [Recall]

What's the main purpose of a `.gitignore` file?

- **A.** To delete files from the remote repository
- **B.** To tell Git which untracked files/patterns to exclude from being staged or committed, keeping build artifacts, secrets, and local config out of version control
- **C.** To prevent other developers from cloning the repository
- **D.** To automatically format code before commit

<details>
<summary>Answer</summary>

**B**

**Why B:** `.gitignore` patterns (e.g. `node_modules/`, `.env`, `__pycache__/`, `*.log`) tell Git to not track matching files even if they exist in the working directory — critical for keeping generated files, local secrets, and IDE config out of shared history.

**Why not A:** `.gitignore` only affects *untracked* files going forward — it has no effect on files already committed and pushed; removing those requires `git rm` (and for secrets specifically, often history rewriting and credential rotation).

**Why not C:** `.gitignore` has no bearing on repository access or clone permissions — that's controlled separately (repo visibility, collaborator permissions).

**Why not D:** Code formatting is handled by separate tooling (formatters, pre-commit hooks) — `.gitignore` only controls what's tracked, not how tracked code is formatted.

**Interviewer's likely follow-up:** *"A secret got committed and pushed three commits ago. Is adding it to `.gitignore` now enough?"* (Answer: no — the secret is already in history and anyone with repo access (or who cloned it) can retrieve it; you need to rotate the actual credential immediately, and separately consider history-rewriting tools like `git filter-repo` if the exposure needs to be scrubbed from history, understanding that doesn't help if it was ever pushed publicly.)
</details>

### Q1.28 · API design · [Design]

You're designing a webhook system so a client's server gets notified when an event (e.g. "booking confirmed") happens on your platform. What's the most important reliability property to build in from the start?

- **A.** Sending the webhook over HTTPS
- **B.** At-least-once delivery with retries and a way for the receiver to deduplicate (e.g. an event ID), since the receiving server might be down, slow, or return an error when you first try
- **C.** Sending webhooks in strict chronological order, always
- **D.** Making the payload as small as possible

<details>
<summary>Answer</summary>

**B**

**Why B:** Networks and receiving servers are unreliable — the receiver might be deploying, down, or briefly erroring when you first attempt delivery. Without retries, events silently vanish for the client. But retries introduce the possibility of the same event being delivered twice (e.g. the first attempt succeeded but the ack was lost), so the receiver needs a stable, unique event ID to deduplicate on their side — this is the same idempotency pattern as Q1.3, now on the sending side.

**Why not A:** HTTPS is a baseline security requirement, but it says nothing about what happens when the receiving endpoint fails to respond — it doesn't address reliability of delivery at all.

**Why not C:** Strict ordering is often desirable but is a much harder guarantee to make under retries and failures (an earlier event might need more retries than a later one), and most real integrations design for "eventually consistent, dedupable" delivery rather than guaranteeing strict order — over-promising this can cause more problems than it solves.

**Why not D:** Payload size is a secondary optimization; a large payload delivered reliably is far more useful than a tiny payload that silently gets lost with no retry.

**Interviewer's likely follow-up:** *"How would the client verify a webhook request actually came from you and wasn't forged?"* (Answer: sign the payload with a shared secret (HMAC), send the signature in a header, and have the client verify it before trusting the payload — the same principle as verifying any inbound request from an untrusted network.)
</details>

### Q1.29 · Error handling / retries · [Applied]

Your service calls a downstream dependency that has been failing 100% of requests for the last two minutes. Your retry logic (with backoff and jitter) is technically working correctly, but the failure rate and latency on *your* service are climbing because every request still waits through several retry attempts before ultimately failing. What pattern addresses this?

- **A.** Increase the number of retries so more requests eventually succeed
- **B.** A circuit breaker — after a failure-rate threshold is crossed, stop attempting calls to the downstream entirely for a cooldown period, failing fast instead of retrying into a known-broken dependency
- **C.** Remove retries entirely
- **D.** Add a longer timeout so requests have more time to succeed

<details>
<summary>Answer</summary>

**B**

**Why B:** Retrying with backoff is the right response to *transient* failure, but once a dependency is reliably down, every retry is wasted latency and load. A circuit breaker tracks the failure rate; once it crosses a threshold, it "opens" and fails fast without even attempting the call, protecting your own service's latency and the downstream from further pressure — then periodically allows a trial request through ("half-open") to detect recovery.

**Why not A:** More retries against a 100%-failing dependency just means more wasted latency per request, worsening the exact symptom described — it doesn't address a sustained (non-transient) outage at all.

**Why not C:** Removing retries entirely would help this specific sustained-outage scenario but would reintroduce failures on genuinely transient blips (the case retries are correctly designed for) — you want both retries for the transient case and a circuit breaker for the sustained case.

**Why not D:** A longer timeout makes each doomed request take even longer before failing, actively worsening your service's latency — the opposite of what's needed when the dependency is reliably down.

**Interviewer's likely follow-up:** *"What does your service do while the circuit is open — just return errors to your own callers?"* (Answer: depends on the use case — ideally degrade gracefully, e.g. serve cached/stale data, a default response, or a clear "temporarily unavailable" error, rather than either hanging or silently pretending everything's fine.)
</details>

### Q1.30 · SQL basics · [Recall]

What does a database transaction's "atomicity" guarantee mean?

- **A.** The transaction runs faster than a non-transactional query
- **B.** All operations within the transaction either all succeed and commit together, or if any part fails, the entire transaction rolls back — leaving no partial changes
- **C.** The transaction is encrypted
- **D.** Only one user can run a transaction at a time

<details>
<summary>Answer</summary>

**B**

**Why B:** Atomicity is the "A" in ACID — it guarantees a transaction is all-or-nothing. If you're transferring money between two accounts (debit one, credit the other) and the credit step fails, atomicity ensures the debit is rolled back too, rather than leaving money deducted from one account with nowhere it went.

**Why not A:** Atomicity says nothing about speed — transactions often add overhead (locking, logging) compared to non-transactional operations; the guarantee is about correctness under failure, not performance.

**Why not C:** Atomicity has nothing to do with encryption — that's a separate concern (data at rest/in transit encryption) entirely unrelated to the ACID properties.

**Why not D:** That describes a concurrency/locking behavior, not atomicity — and it's not accurate either; databases support many concurrent transactions, managed through isolation levels (the "I" in ACID), a related but distinct property.

**Interviewer's likely follow-up:** *"You update a user's profile and, in the same request, log an analytics event to a separate analytics database. Should both be in one transaction?"* (Answer: usually no — most databases can't span a single ACID transaction across two separate database systems; the common pattern is to make the critical write (profile update) transactional within its own database, and handle the analytics event as a best-effort side effect, or use an outbox/event pattern if you need stronger delivery guarantees across systems.)
</details>

---

## Explain prompts

### E1.1 · Explain: why exponential backoff with jitter, not fixed-delay retries

**Prompt:** *"Walk me through why you'd use exponential backoff with jitter for retrying failed API calls, instead of just retrying every 2 seconds."*

**Target:** 60–90 seconds spoken. Answer out loud before opening the rubric.

<details>
<summary>Rubric — score yourself</summary>

**Must hit (1 point each):**
- [ ] States that fixed-delay retries from many clients synchronize into repeated bursts of load on an already-struggling service
- [ ] Explains exponential backoff: delay grows with each attempt (1s, 2s, 4s...), giving the downstream progressively more time to recover
- [ ] Explains jitter: adding randomness to the delay desynchronizes clients that failed at the same moment, spreading retries instead of bursting them
- [ ] Notes there should be a cap on both delay and total retry count/time — unbounded retries can hang a request indefinitely
- [ ] Mentions this matters most at scale — with one client it barely matters, with thousands of clients hitting the same failure it's the difference between recovery and a sustained outage

**Bonus — signals strength:**
- [ ] Distinguishes "full jitter" vs "equal jitter" or describes a concrete jitter formula
- [ ] Mentions respecting a `Retry-After` header when the server provides one
- [ ] Connects it to circuit breakers as the next escalation when retries alone aren't enough

**Red flags — deduct:**
- [ ] Says jitter is "just to make it random" with no explanation of *why* randomness helps
- [ ] Can't explain what problem fixed-delay retries actually cause
- [ ] Suggests retrying forever with no cap

**Score: ___ / 5**

**Model answer:**
"So the problem with just retrying every 2 seconds is — if a downstream service has a blip and a thousand clients all failed around the same time, they all retry at exactly 2 seconds, then all fail again together, then all retry at 4 seconds together. You get these synchronized waves of load hitting the service right when it's trying to recover, which can actually keep it down longer. Exponential backoff helps a bit — the delay grows each time, 1, 2, 4, 8 seconds — but on its own it's still synchronized. Jitter is what actually breaks the synchronization — you add randomness to each client's delay so instead of a wave, you get retries spread out over a window. And you'd cap it — max delay, max attempts — so a single request doesn't just hang forever if the thing's genuinely broken."
</details>

### E1.2 · Explain: how you'd design idempotent retries for a payment call

**Prompt:** *"You're integrating with a payment provider's API to charge a customer. Your call to charge them times out — you don't know if it went through. Walk me through how you'd make retrying this safe."*

**Target:** 60–90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit (1 point each):**
- [ ] Identifies the core risk: retrying a timed-out charge could double-charge the customer if the original request actually succeeded server-side
- [ ] Names the idempotency key pattern — generate a unique key client-side per logical charge attempt, send it every time including retries
- [ ] Explains the server-side behavior needed: the provider stores the key with the result and returns the cached result on a repeat, instead of re-executing
- [ ] States this key must be the same across retries of the same logical operation (not regenerated each retry) but different for genuinely separate charges
- [ ] Mentions checking the actual outcome (e.g. querying transaction status) before blindly retrying is also a valid complementary step

**Bonus — signals strength:**
- [ ] Notes that most real payment providers (Stripe, etc.) support this natively via an `Idempotency-Key` header
- [ ] Discusses what to do if the provider doesn't support idempotency keys natively — e.g. check-then-charge with your own ledger
- [ ] Mentions a reasonable TTL/expiry for stored idempotency keys server-side

**Red flags — deduct:**
- [ ] Suggests just retrying the exact same request with no idempotency mechanism at all
- [ ] Doesn't recognize double-charging as the actual risk being asked about
- [ ] Proposes a fix that requires the customer to manually confirm before every retry

**Score: ___ / 5**

**Model answer:**
"The scary part here isn't that the request failed — it's that I don't know *whether* it failed. It might have gone through on their end and I just didn't get the response back. So blindly retrying the exact same charge risks double-charging the customer. The standard fix is an idempotency key — I generate a unique key for this specific charge attempt before I even make the first call, and I send that same key on every retry. On their side, if they see that key again, they don't re-run the charge — they just return whatever happened the first time. Most payment providers support this out of the box, it's literally a header. If for some reason they didn't, I'd want to check the actual transaction status before retrying — query 'did this go through' rather than just firing the same request blind."
</details>

### E1.3 · Explain: unit test vs integration test tradeoffs for LLM-calling code

**Prompt:** *"You're building a feature that calls an LLM API as part of a larger pipeline. How do you think about what to unit test versus what needs an integration test?"*

**Target:** 60–90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit (1 point each):**
- [ ] Identifies that the LLM call itself should be mocked for unit tests of surrounding logic (prompt construction, parsing, error handling, retry behavior)
- [ ] States that unit tests with a mocked LLM give fast, deterministic coverage of your own code's correctness
- [ ] Recognizes mocks can drift from the real API's actual behavior/schema over time, and that's a real risk, not a hypothetical one
- [ ] Proposes a smaller set of real (or semi-real, e.g. cached real responses) integration tests to catch that drift and validate actual output quality
- [ ] Mentions non-determinism specifically — integration tests against a real LLM need property-based or semantic assertions, not exact string matches

**Bonus — signals strength:**
- [ ] Distinguishes testing *your code's logic* from testing *the model's output quality* — the latter is closer to evaluation than testing (see file 06)
- [ ] Mentions running the smaller integration suite less frequently (e.g. nightly, not on every commit) to avoid slowing down CI or introducing flakiness into the fast feedback loop
- [ ] Notes cost as a practical reason not to run real LLM calls on every PR

**Red flags — deduct:**
- [ ] Says you should just mock everything, full stop, with no plan for catching drift
- [ ] Says you should never mock and always call the real API in every test
- [ ] Tries to assert exact string equality on real LLM output without acknowledging non-determinism

**Score: ___ / 5**

**Model answer:**
"For the code around the LLM call — how I build the prompt, how I parse the response, what happens if the call errors out — I'd mock the LLM client and unit test that logic directly, fast and deterministic. But mocks are only as good as the assumption they encode about what the real API returns, and that can drift — the provider changes a response field, or my prompt starts producing a slightly different output shape, and my mock wouldn't catch that. So I'd also keep a smaller set of tests that hit the real API, but I can't assert exact string equality because the output isn't deterministic even at temperature zero sometimes — so those tests check structural things instead, valid JSON, required fields present, length within bounds, maybe a semantic similarity check against a reference. And I'd run those less often, not on every single commit, partly for cost and partly so a model hiccup doesn't block someone's unrelated PR."
</details>

### E1.4 · Explain: async vs multiprocessing

**Prompt:** *"When would you reach for `asyncio` versus `multiprocessing` in Python, and why not just always use one?"*

**Target:** 60–90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit (1 point each):**
- [ ] States asyncio is for I/O-bound work — waiting on network calls, disk, database — where the CPU is idle during the wait
- [ ] States multiprocessing is for CPU-bound work — actual computation that keeps a core busy the whole time
- [ ] Mentions the GIL (Global Interpreter Lock) as the reason threads/async don't give you true CPU parallelism in standard Python
- [ ] Explains why "always use one": asyncio on CPU-bound work blocks the event loop (nothing else runs while the CPU-heavy function executes); multiprocessing on I/O-bound work wastes resources — process overhead for something that's mostly waiting, not computing
- [ ] Gives or implies a concrete example of each category

**Bonus — signals strength:**
- [ ] Mentions you can combine them — offload CPU-bound work to a process pool executor from within an async application
- [ ] Notes multiprocessing has real overhead (process startup, serialization to pass data between processes) that async doesn't have
- [ ] Mentions threading as a third option and where it still has a place (e.g. some I/O-bound C-extension libraries release the GIL)

**Red flags — deduct:**
- [ ] Says async and multiprocessing are basically interchangeable
- [ ] Can't explain what the GIL is or why it matters here
- [ ] Recommends multiprocessing for a clearly I/O-bound scenario or vice versa

**Score: ___ / 5**

**Model answer:**
"It really comes down to what your program is actually waiting on. If it's waiting on I/O — an API call, a database query, reading a file — the CPU's just sitting idle during that wait, and asyncio lets you use that idle time to make progress on other tasks, all on one thread. But if the work is actually CPU-bound, like crunching numbers or resizing images, asyncio doesn't help at all, because Python's GIL means only one thread runs Python bytecode at a time anyway — so if you `await` something that's actually just burning CPU, you block the whole event loop, nothing else can run. That's when you want multiprocessing, separate processes, each with its own interpreter and GIL, actually running on different cores in parallel. You wouldn't want to always use multiprocessing either, though — spinning up processes has real overhead, and for something that's 95% waiting on a network call, that overhead is pure waste."
</details>

### E1.5 · Explain: idempotency, safety, and why it matters for retries

**Prompt:** *"Explain the difference between a 'safe' HTTP method and an 'idempotent' one, and why that distinction actually matters when you're writing retry logic."*

**Target:** 60–90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit (1 point each):**
- [ ] Defines safe: no side effects on server state (e.g. `GET`)
- [ ] Defines idempotent: repeating the same request produces the same end state as doing it once, even if it does have side effects (e.g. `PUT`)
- [ ] States `POST` is neither safe nor idempotent by default
- [ ] Connects this to retries: it's generally safe to auto-retry safe or idempotent operations, but risky to blindly retry a non-idempotent one (like a default `POST`) without extra protection
- [ ] Mentions the idempotency-key pattern as how you make a `POST`-style operation safely retryable

**Bonus — signals strength:**
- [ ] Correctly classifies `DELETE` as idempotent-but-not-safe
- [ ] Notes `PATCH` idempotency depends on the specific semantics of the patch operation
- [ ] Gives a concrete failure example (double-charge, duplicate order) to ground the explanation

**Red flags — deduct:**
- [ ] Uses "safe" and "idempotent" interchangeably
- [ ] Says all HTTP methods are safe to retry automatically
- [ ] Can't connect the concept back to a concrete retry-logic consequence

**Score: ___ / 5**

**Model answer:**
"Safe means no side effects at all — a `GET` shouldn't change anything on the server, so you can call it as many times as you want with zero risk. Idempotent is a weaker guarantee — it *can* have a side effect, but doing it once versus doing it five times in a row leaves the system in the same end state. `PUT` is like that — if I `PUT` the same full update five times, the resource ends up the same either way. `POST` is usually neither, by default — if I `POST` an order five times, I've probably created five orders. This matters for retries because it's basically free to auto-retry a `GET` or `PUT`, but auto-retrying a raw `POST` is dangerous — that's exactly the double-charge, duplicate-order kind of bug. So if you need retryable `POST`-like behavior, you make it idempotent yourself, usually with an idempotency key the server can use to recognize 'oh, I already did this one.'"
</details>

### E1.6 · Explain: what a good code review catches beyond "does it work"

**Prompt:** *"Beyond checking that the code works, what are you actually looking for when you review a teammate's pull request?"*

**Target:** 60–90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit (1 point each):**
- [ ] Mentions checking whether tests actually assert meaningful behavior, not just that they exist
- [ ] Mentions error handling — are failure modes considered, or does the happy path assume everything succeeds
- [ ] Mentions readability/maintainability for someone who isn't the author, six months from now
- [ ] Mentions whether the change's scope matches its stated purpose — no unrelated changes bundled in
- [ ] Mentions edge cases specific to the domain (e.g. concurrent access, empty inputs, non-deterministic external calls)

**Bonus — signals strength:**
- [ ] Distinguishes blocking feedback from nitpicks/preferences, and signals that distinction in review comments
- [ ] Mentions security implications (input validation, secrets, injection) as part of the review lens
- [ ] Mentions checking that the PR is reviewable in the first place — size/scope appropriate for a human to actually evaluate carefully

**Red flags — deduct:**
- [ ] Only mentions style/formatting as the main thing being checked
- [ ] Says they just check that tests pass in CI and approve
- [ ] Can't name any specific failure mode or edge case they look for

**Score: ___ / 5**

**Model answer:**
"Whether it works is honestly the baseline, not the review. What I'm actually checking is — do the tests prove it works, or do they just exist? I'll look at the assertions specifically, not just whether there's a test file. Then error handling — what happens when the API call fails, when the input's empty, when two requests hit this at the same time — because that's the stuff that doesn't show up in the demo but shows up at 2am in production. I'm also thinking about the next person reading this code, not just whether it's correct today — is it going to be obvious what this does in six months. And scope — if a PR titled 'fix pagination bug' also refactors three unrelated files, that's a red flag, because now I can't review either change properly, and if something breaks I don't know which part caused it."
</details>

### E1.7 · Explain: how you'd diagnose a slow SQL query

**Prompt:** *"A query that used to run in 50ms is now taking 4 seconds. Nothing in the query text changed. Walk me through how you'd figure out why."*

**Target:** 60–90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit (1 point each):**
- [ ] Starts with `EXPLAIN` / `EXPLAIN ANALYZE` to see the actual query plan being used now, not guessing
- [ ] Considers data growth as a likely cause — the table may have grown enough that a previously-fine plan (or missing index) now matters
- [ ] Considers whether an index was dropped, is unused by the planner now, or statistics are stale (causing the planner to pick a bad plan)
- [ ] Considers external factors — locking/contention from other queries, or a change in server load/resources, not just the query itself
- [ ] Proposes a concrete next step based on what the plan shows, not just "add an index" reflexively

**Bonus — signals strength:**
- [ ] Mentions checking if a sequential scan replaced what used to be an index scan
- [ ] Mentions `ANALYZE` (or equivalent) to refresh planner statistics as a low-risk first thing to try
- [ ] Mentions checking recent schema/migration history even though "the query text" didn't change — surrounding schema might have

**Red flags — deduct:**
- [ ] Jumps straight to "add an index" without looking at the plan first
- [ ] Assumes it must be a code bug without considering data growth or infrastructure
- [ ] Has no concrete first diagnostic step, just guesses

**Score: ___ / 5**

**Model answer:**
"First thing I'd do, before touching anything, is run `EXPLAIN ANALYZE` on it and actually look at the plan the database is using right now, not assume. If I see a sequential scan where I'd expect an index scan, that tells me something — either an index got dropped, or the table's grown enough that the row count crossed some threshold where the planner's decisions look different, or the planner's statistics are just stale and it's making a bad call. Since the query text itself didn't change, my first suspects are data volume and the schema around it, not the query — did the table 10x in size, did someone drop an index recently, was there a migration. I'd also check if something else is contending for locks on that table around the same time. But I wouldn't just reflexively add an index — I'd want the plan to actually tell me that's the fix before I commit to it."
</details>

### E1.8 · Explain: why you'd choose cursor-based over offset-based pagination

**Prompt:** *"You're designing pagination for an endpoint that lists a high-write-volume table — new rows are constantly being inserted. Would you use offset or cursor-based pagination, and why?"*

**Target:** 60–90 seconds spoken.

<details>
<summary>Rubric — score yourself</summary>

**Must hit (1 point each):**
- [ ] Chooses cursor-based (keyset) pagination for this scenario
- [ ] Explains offset pagination's failure mode: row positions shift under concurrent inserts/deletes, causing skipped or duplicated rows across pages
- [ ] Explains cursor pagination's mechanism: paginate on a stable, unique, ordered key (e.g. `id` or `created_at + id`), fetching rows after the last-seen cursor value
- [ ] States cursor pagination is stable under concurrent writes because it doesn't depend on absolute row position
- [ ] Acknowledges a real tradeoff of cursor pagination (e.g. can't jump to an arbitrary page number, only "next")

**Bonus — signals strength:**
- [ ] Mentions offset pagination gets slower on later pages too (the database still has to count/skip N rows), a performance issue independent of the correctness issue
- [ ] Mentions needing a composite cursor (not just one column) if the sort key isn't already unique on its own
- [ ] Notes offset pagination is still fine for smaller, low-write, or admin-tool-style tables where simplicity outweighs this risk

**Red flags — deduct:**
- [ ] Picks offset pagination for a high-write table without acknowledging the risk
- [ ] Can't explain *why* offset pagination breaks under concurrent writes
- [ ] Describes cursor pagination incorrectly (e.g. as just "a different name for offset")

**Score: ___ / 5**

**Model answer:**
"I'd go cursor-based here. Offset pagination works by literally re-running the query with 'skip N rows' every time you fetch a page — but if rows are constantly being inserted, what row is at position 1000 keeps changing between your page-1 and page-2 requests. So users end up seeing duplicates or missing rows entirely, and it's confusing because nothing looks broken from the data's perspective, it's a pagination artifact. Cursor pagination instead says 'give me everything after this specific ID,' using a stable, ordered column — so no matter what gets inserted elsewhere, your next page is defined relative to a fixed point, not a shifting position. The real tradeoff is you lose the ability to jump straight to 'page 47' — you can only go forward from a cursor — but for a feed-style or high-write table, that's usually a reasonable price for actually correct results."
</details>
