# Interview Q&A

## Architecture & Design

**Q: Walk me through how your crawler is structured.**

There are four layers. `scraper.ts` handles a single URL — it fetches, retries on transient errors, checks the post-redirect host, validates the content type, and extracts links. It returns a `CrawlResult` with no knowledge of where it sits in a crawl. `crawl.ts` is the orchestrator — it manages the BFS queue, the `visited` dedup set, concurrency via an `inFlight` set, and calls `scraper` for each URL. `normalizeUrl.ts` is a pure utility that canonicalises URLs so the same logical page doesn't get visited twice under different representations. `index.ts` wires config from environment variables and provides the `onCrawlResult` callback that prints output. Each layer has one job and can be tested independently.

---

**Q: Why did you use a callback (`onCrawlResult`) rather than returning a list of results at the end?**

Two reasons. First, memory — a large site could have thousands of pages, and accumulating all results before printing would hold everything in memory. A callback streams results as they arrive. Second, testability — in tests I can pass a simple collector function and assert on intermediate state without changing `crawl.ts` at all. Returning a list would couple the crawler to a specific output shape and make it harder to swap in different consumers.

---

**Q: Why did you choose BFS over DFS?**

BFS finds shallow pages first, which means you start printing results quickly and the `maxDepth` limit has an intuitive meaning — depth 1 is pages directly linked from the start page. DFS would dive into one branch entirely before exploring others, which could exhaust `maxDepth` on a single chain and miss large sections of the site. BFS also distributes work more evenly across the site, which pairs better with a concurrency limit.

---

**Q: How does `maxDepth` work? What does depth 1 mean?**

The start URL is depth 0. Every link found on a depth-0 page is depth 1, and so on. When `processQueueItem` runs a page at depth `N` and `N >= maxDepth`, it scrapes the page and calls the callback, but doesn't enqueue any of the links it found. So `maxDepth: 1` visits the start URL and all pages directly linked from it, but stops there. If `maxDepth` is `undefined`, there's no limit. The check is `options.maxDepth !== undefined && depth >= options.maxDepth` — using a strict `!== undefined` rather than a falsy check means `maxDepth: 0` correctly limits the crawl to just the start URL.

---

## Concurrency

**Q: Explain how your concurrency model works.**

I maintain an `inFlight` Set of promises. The outer loop runs as long as there's work queued or in flight. The inner loop greedily dequeues items and calls `startTask` until `inFlight.size` reaches `maxConcurrency`. Then I call `await Promise.race(inFlight)` which suspends until the fastest in-flight task completes, freeing a slot. At that point the `.finally()` on the completed task has already removed it from `inFlight`, so the outer loop immediately starts the next item. This means I always have up to `maxConcurrency` fetches running simultaneously without ever over-committing.

---

**Q: Why async/await rather than worker threads?**

The crawler is I/O-bound — the bottleneck is waiting for network responses, not doing computation. Node's event loop handles concurrent `fetch()` calls natively; no threads are needed. In an earlier version I used `worker_threads` where each worker spawned, fetched one URL, and terminated. The benchmarks show that was 4× slower because you pay the full OS thread lifecycle — spawn, JIT warmup, teardown — for every single URL. For CPU-bound work like heavy HTML processing or running a headless browser, threads would be justified. For `fetch` + DOM traversal, async/await is both simpler and faster.

---

**Q: Is there a race condition on the `visited` set with concurrent tasks?**

No. JavaScript is single-threaded — there's no true parallelism, only interleaved async execution. The link-processing loop inside `processQueueItem` (`for (const link of result.links)`) is synchronous. It runs to completion as a microtask before any other async continuation can execute. So if two tasks both discover a link to `/foo`, the first one to complete its `await scrape()` will synchronously add `/foo` to `visited` and push it to the queue, and by the time the second task's continuation runs, `visited.has('/foo')` will be true and it will skip it.

---

**Q: Could your `Promise.race` loop ever deadlock or terminate prematurely?**

No. The outer while condition is `queue.length > 0 || inFlight.size > 0`, so we only exit when both are empty. New URLs are added to `queue` synchronously inside `processQueueItem` before the function returns, which means before the `.finally()` that removes the task from `inFlight`, which means before `Promise.race` propagates its resolve to the outer loop. So when the outer loop checks the queue after each race, it always sees the complete picture. The only way to exit early would be if `processQueueItem` threw unhandled, which can't happen because `scrape` catches all errors internally and the `onCrawlResult` callback is wrapped in a try/catch.

--- 
**Q: Could two concurrent fetches visit the same URL twice?** 
Yes — there's a TOCTOU race. `visited.add(link)` happens inside `processQueueItem`, which runs concurrently. If two in-flight tasks both discover the same link and both call `visited.has()` before either calls `visited.add()`, they both pass the check and both enqueue it. The fix is to claim the URL at the moment it's first seen, in the driver loop, before any task is dispatched: 

```typescript
for (const link of links) { 
  if (!visited.has(link)) { 
    visited.add(link);  // claim before enqueue 
    queue.push({ url: link, depth: depth + 1 }); 
  } 
} 
```
This works because the driver loop runs on the single JS event loop thread — the for loop itself is synchronous, so there's no interleaving between has() and add().

---

**Q: Why didn't your tests catch this?**
All tests run with maxConcurrency: 1, so tasks never interleave.
To surface the race I'd add a test with maxConcurrency: 5 where multiple mocked pages return the same overlapping link, then assert it's only visited once. 

---
**Q: Why is processQueueItem defined inside crawl()?**
What's the downside? It's a closure over queue, visited, and options — convenient because it avoids threading those through as parameters. The downside is it can't be imported and unit tested in isolation; any test of it is implicitly an integration test of the whole crawl loop. At this scale it's fine. If the per-URL logic grew more complex — robots.txt checks, priority scoring, per-host rate limiting — I'd extract it into a named module function and pass dependencies explicitly, which also makes them mockable.

---
**Q: What's missing from your test suite?**
A few gaps I'd prioritise:
1. Concurrent deduplication — a test with maxConcurrency > 1 and overlapping links across pages, asserting each URL is visited exactly once. This would have caught the race above.
2. Scraper edge cases — the redirect-out-of-scope path and non-HTML content-type path both have code but no tests; straightforward to add with nock.
3. Fetch timeout — AbortSignal.timeout(5000) is untested. I'd either mock a hanging response or make the timeout configurable so it can be injected in tests.
4. env.ts — no coverage for missing or invalid env values like NaN for MAX_CONCURRENCY. That's a real failure mode since it's the user-facing entry point.

---

## URL Handling

**Q: Walk me through how you prevent visiting the same page twice.**

The `visited` Set holds normalised URL strings. Before enqueueing any link, I check `visited.has(link)` and add it immediately if not present. Normalisation is the critical part — without it, `https://crawlme.monzo.com/about`, `https://crawlme.monzo.com/about/`, `https://CRAWLME.MONZO.COM/about`, and `https://crawlme.monzo.com:443/about` are four different strings but the same logical page. `normalizeUrl` handles all of these: it lowercases the hostname, strips `www.`, removes default ports, strips trailing slashes on non-root paths, and drops query strings and fragments.

---

**Q: Why do you strip query strings?**

To prevent infinite crawl spirals. Sites commonly append tracking or pagination parameters — `?ref=nav`, `?page=2`, `?session=xyz` — and if you treat each as a distinct URL you'd loop forever or visit thousands of variations of the same page. Stripping them collapses all variants to the canonical path. The trade-off is that pages that are genuinely differentiated by query string — like `/search?q=typescript` vs `/search?q=python` — get de-duplicated to `/search`. For this problem that's acceptable, but a general-purpose crawler would need this to be configurable.

---

**Q: How do you handle redirects that leave the allowed subdomain?**

The `fetch` call in `scraper.ts` uses `redirect: 'follow'`, so the browser follows the chain automatically. After the response comes back, I read `res.url` which is the final URL after all redirects, normalise its hostname, and compare it against `allowedHost`. If they don't match, I set `result.error` and return empty links — the page is recorded as visited but nothing it links to will be followed. This correctly handles cases like a page on `crawlme.monzo.com` that 301s to `monzo.com`.

---

## Error Handling & Resilience

**Q: How does your retry logic work and what does it retry?**

I use `exponential-backoff` with full jitter and up to 5 attempts, with a maximum delay of 10 seconds. Inside the backoff callback I throw on `status >= 500` or `status === 429` — those are the retryable conditions, meaning the server is overloaded or rate-limiting me. Non-retryable responses like 404 or 403 don't throw; they return the response normally and the `if (!res.ok)` check records an error and continues. Network errors and timeouts (from `AbortSignal.timeout(5000)`) also propagate as exceptions and get retried. Full jitter matters here — without it, all concurrent crawlers would back off to the same interval and slam the server simultaneously when they retry.

---

**Q: What happens when a page returns a non-200 status or times out? Does the crawler stop?**

No, the crawler continues. `scraper.ts` wraps everything in a try/catch and always returns a `CrawlResult`. If there's an error — network failure, timeout, bad status after retries — it's recorded in `result.error` and `result.links` is empty. The callback still fires so the error is visible in output, but since `links` is empty there's nothing to enqueue. The crawl carries on with the remaining queue.

---

**Q: What if the `onCrawlResult` callback throws?**

It's wrapped in a try/catch in `processQueueItem`. Before that guard existed, a throwing callback would propagate through the `.finally()` promise and reject `Promise.race`, crashing the entire crawl. Now the error is logged and the crawler continues. The contract is that the callback is best-effort — a bad consumer can't take down the crawl.

---

## Testing

**Q: How did you test the crawler without hitting real websites?**

I used `nock`, which intercepts Node's HTTP layer and lets you define exact responses for specific URLs. The tests in `crawl.spec.ts` set up fake HTML responses, call `crawl()` with a collector callback, and assert on which URLs were visited and what links were found. This means tests run instantly, are deterministic, and don't depend on network availability. `nock.cleanAll()` runs in `afterEach` to reset state between tests.

---

**Q: What would you test that you haven't tested yet?**

A few gaps. The retry-on-429 path — the scraper tests cover 5xx but not rate-limiting specifically. Concurrent behaviour under `maxConcurrency > 1` — all crawl tests use `maxConcurrency: 1` so they're effectively sequential. The `maxDepth: 0` edge case — crawl only the start URL. And an integration test against a local HTTP server (rather than nock) would give higher confidence that the full stack hangs together correctly, including real redirect chains.

---

## Scalability & Production

**Q: If a site has thousands of pages, how would your crawler handle that?**

The crawler handles it correctly by design. The BFS queue and `visited` Set grow as links are discovered, but there's no assumption about site size — the loop runs until both are empty regardless of how many pages that takes. `maxConcurrency` keeps the number of simultaneous fetches bounded so the process doesn't open thousands of connections at once. `maxDepth` can cap how deep the crawl goes if you want to limit scope on a large site.

The main practical concern at thousands of pages is time, not correctness. At `maxConcurrency: 10` with an average response time of 200ms, crawling 10,000 pages takes roughly 200 seconds. Tuning `maxConcurrency` up (within what the server tolerates) is the primary lever. Beyond that, the exponential backoff on 5xx/429 responses means the crawler self-regulates if the server starts pushing back — it slows down automatically rather than hammering a struggling server.

`maxDepth` is a useful but blunt instrument here. It bounds by graph depth, not by page count — a shallow but wide site could have thousands of pages all at depth 1, so `maxDepth: 2` gives you no protection against visiting tens of thousands of pages. If you need a hard page-count cap, you'd need a separate `maxPages` limit that stops enqueueing once a threshold is reached. The two controls solve different problems: `maxDepth` shapes the crawl by site structure, `maxPages` shapes it by volume.

Memory is fine at this scale. Thousands of URLs at ~50 bytes each is a few megabytes. The in-memory `visited` Set only becomes a concern at millions of pages.

---

**Q: What breaks first if you point this at a site with a million pages?**

The `visited` Set. It holds every URL seen as an in-memory string, growing without bound. A million URLs at ~50 bytes each is ~50MB — manageable — but beyond that you'd need a persistent store. Redis with a set would keep memory flat and also let you pause and resume the crawl. SQLite would work for a single-machine crawler that needs durability without an external dependency.

---

**Q: What else would you add before calling this production-ready?**

Three things. `robots.txt` support — a production crawler should fetch and respect `/robots.txt` before crawling, both for legal/ToS reasons and basic politeness. Per-host rate limiting — `maxConcurrency` bounds parallelism but not request rate; a token bucket or minimum inter-request delay would prevent hammering a server. Graceful shutdown — a `SIGINT` handler that stops dequeuing and awaits `Promise.all(inFlight)` so in-flight connections drain cleanly rather than being abandoned on Ctrl+C.

---

**Q: How would you scale this to crawl multiple domains simultaneously?**

The current `crawl` function is scoped to one `allowedHost`. To crawl multiple domains you'd either run multiple independent `crawl()` calls in parallel (simplest), or generalise the orchestrator to manage per-domain queues and a shared concurrency budget. The `scraper` layer already has no domain-level state so it'd work unchanged. The main concern would be fairness — a global concurrency limit could be monopolised by one fast domain, so you'd want per-domain concurrency caps.

---

**Q: How would you refactor `depth` if `CrawlResult` was consumed in more places?**

Split into two types at the layer boundary. `scraper.ts` genuinely doesn't know about depth — it handles one URL in isolation. `crawl.ts` is what adds traversal context. So:

```typescript
// ScrapeResult — returned by scraper.ts
type ScrapeResult = {
  url: string;
  links: string[];
  error?: string;
};

// CrawlResult — emitted by crawl.ts to consumers
type CrawlResult = ScrapeResult & {
  depth: number; // required, not optional
};
```

`scraper.ts` returns `ScrapeResult`. `crawl.ts` promotes it to `CrawlResult` by adding `depth` before calling `onCrawlResult`. Every consumer then gets a type-system guarantee that `depth` is always present — no defensive `!== undefined` checks needed. The optional was a leaky abstraction: `depth` isn't part of fetching a page, it's part of orchestrating a crawl, so it belongs on the type that represents the crawl-layer result.

---                                                                                                                                                            
Q: You added maxDepth and maxConcurrency — neither was required. Why?                                                                                          
                                                                                                                                                               
A: maxConcurrency is a safety mechanism, not a feature. Without it, the crawler fires an unbounded number of concurrent HTTP requests — one per discovered link. Against a large site this can exhaust file descriptors, overwhelm the target server, or get the client IP rate-limited or banned. A bounded concurrency pool (defaulting to 5) makes the crawler safe to run without coordination overhead. The implementation uses a Set<Promise> + Promise.race so throughput stays high — we never stall waiting for slow requests when there are pending items in the queue.

maxDepth is a circuit breaker against infinite or cyclical graphs. Web graphs aren't trees — sites can form link cycles that make unbounded BFS loop forever (or until memory is exhausted). Even with the visited-set guard, a deeply interlinked site with millions of unique pages could run indefinitely. maxDepth gives the caller a hard stop. It defaults to undefined (infinite) so the base behavior is unchanged — it's an opt-in escape hatch, not a restriction.               

Tradeoffs:
|                      | maxConcurrency                                                                 | maxDepth                                      |
|----------------------|---------------------------------------------------------------------------------|-----------------------------------------------|
| Problem it solves    | Unbounded parallel requests exhaust file descriptors and hammer the target     | Cyclic/deep graphs cause unbounded runtime and memory growth |
| Benefit              | Predictable resource usage; respects rate limits                               | Bounded runtime; predictable memory ceiling   |
| Cost                 | Lower raw throughput vs. fully parallel                                        | May miss content at deeper levels             |
| Default behavior     | 5 (safe out of the box)                                                        | undefined (infinite — opt-in restriction)     |
| What breaks without it | File descriptor exhaustion, IP bans, OOM                                     | Infinite crawl on cyclic graphs               |

Follow-up the interviewer might ask: "Why not use p-limit instead of rolling your own concurrency pool?"

Rolling it with Set + Promise.race avoids a dependency and makes the scheduler visible in the code — the reviewer can see exactly when tasks are drained and how backpressure works. p-limit is a fine choice in production, but in an interview context it obscures the mechanism.       
