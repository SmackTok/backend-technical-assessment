# Senior Backend Engineer — Technical Assessment

Please read this entire document before you start coding. It is long on purpose: the constraints
are part of the exercise.

This is the backend companion to our iOS assessment. Same spirit — a real problem, a real
upstream, honest AI disclosure, and a code-review challenge. The task is shaped around a two-tier,
contract-first design: **a gRPC service that owns the business logic, and a thin HTTP edge that
owns nothing.**

## 1. The Brief

Build a small **Catalog Explorer** backend on top of a real, public, no-auth API: the [Open
Library API](https://openlibrary.org/developers/api).

We picked this over the usual echo-service / URL-shortener / to-do CRUD because it has the
real-world properties we want to see you handle: **pagination, partial and missing fields, an
upstream that is slow and occasionally fails, rate limits, and a non-trivial fan-out.** There is
no API key and no sign-up.

### Shape of the system

Two deployables:

1. A **`CatalogService` gRPC microservice** that owns *all* business logic, upstream calls,
   caching, and resilience.
2. A **thin HTTP edge** that translates each REST route into exactly one gRPC call. It holds no
   business logic and no state. Write the edge in Node/TypeScript to stay in one language, or in
   Go if you prefer — pick one and justify it.

### Upstream endpoints you'll need

```
# Search (paginated). Returns numFound, start, and a docs[] array.
GET https://openlibrary.org/search.json?q=dune&page=1&limit=20&fields=key,title,author_name,first_publish_year,cover_i,edition_count,language

# Work detail
GET https://openlibrary.org/works/{workId}.json

# Editions of a work (paginated)
GET https://openlibrary.org/works/{workId}/editions.json?limit=50

# Edition detail
GET https://openlibrary.org/books/{editionId}.json

# Cover image — only if cover_i is present and >= 0
https://covers.openlibrary.org/b/id/{cover_i}-M.jpg
```

Inspect the live responses — reading an unfamiliar API is part of the exercise. A few things you
will find if you look: `author_name` is frequently absent, `cover_i` is frequently absent, the
work `covers` array contains a `-1` sentinel for "no image", and `description` comes back as a
plain string *sometimes* and as a `{ type, value }` object *other times*. Handle it or document
why you didn't.

## 2. Scope: Core vs. Stretch

> **Do the core first and do it well.** A complete, well-tested, well-architected core beats a
> half-finished pile of stretch goals. We would rather see the core with strong tests than every
> feature with none.

### Core

1. **Contract first** — write the `.proto` before the implementation. A `CatalogService` with at
   least:
   - `Search(SearchRequest) returns (SearchResponse)`
   - `GetWork(GetWorkRequest) returns (WorkResponse)`

   Define a shared `Error { string type; string message; }` and wrap every response in an
   envelope: `{ int32 status; Error error; <payload> data; }`. The envelope is part of the
   contract — the edge must not introduce a second response shape. Generated code is committed
   and never hand-edited — change the `.proto` and regenerate.

2. **Service** — a NestJS gRPC microservice (Node 20, TypeScript) implementing the contract. All
   normalization, caching, resilience, and upstream I/O live here. The upstream must sit behind
   an **interface** you can stub, so the service is testable without the network.

3. **Thin HTTP edge** — a REST surface over the service, one route = one RPC:
   - `GET /v1/catalog/search?q=&page=&limit=`
   - `GET /v1/catalog/works/:id`
   - `GET /v1/catalog/works/:id/editions`
   - `GET /v1/catalog/public/check-health`

   No business logic here. Map gRPC errors onto HTTP statuses and keep the response envelope.

4. **Pagination + search** — paginate search results and edition lists. Your choice of
   page/offset or cursor; justify it and define what "next page" means at the last page.

5. **States & errors** — there is no blank screen here, but there is a blank response. Explicitly
   distinguish: **empty result** (valid, `200`), **upstream not found** (→ `NOT_FOUND`),
   **upstream failure / timeout** (→ `UNAVAILABLE` / `5xx`), and **invalid input** (→
   `INVALID_ARGUMENT` / `400`). A failed upstream call must never come back as a successful empty
   result, and must never hang forever.

6. **Caching & resilience** — put the cache behind an interface (in-memory is fine; Redis is
   better). It must have a TTL, and you should think about **negative caching** (don't stampede the
   upstream for a work that doesn't exist) and **request coalescing** (N concurrent identical
   misses = 1 upstream call, not N). Add a **per-request timeout/deadline**, and **retry with
   backoff + jitter** for idempotent GETs only.

7. **Concurrency** — the work→editions path is a fan-out. Bound it (a work can have hundreds of
   editions), tolerate **partial failure** (one bad edition must not fail the whole request), and
   propagate cancellation/deadlines. No unbounded `Promise.all` over user-controlled sizes.

8. **Tests (TDD — see §3)** — unit tests for the service layer with a **stubbed upstream**, run
   with **no network**, covering at minimum: a successful search, an empty result, an upstream
   error, an upstream timeout, a cache hit after a miss, pagination advancing correctly, and
   normalization of a partial/missing-field record. An integration test that boots the real gRPC
   transport is a plus, not a requirement.

9. **Health** — serve the standard `grpc.health.v1.Health` check so the service is observable.

### Stretch (optional, only after core is solid — pick what interests you)

- **Persistence** — bookmarks/favorites or recent searches in Postgres behind a repository
  interface, with an **additive-only** migration (assume old instances are still serving).
- **Per-client rate limiting** on the edge.
- **Idempotency / request dedup** for the write path if you added one.
- **Observability** — structured logs, a couple of meaningful metrics, and propagating a trace/
  request id from the edge into the service and out to the upstream.
- **Circuit breaker** on the upstream client.
- **`docker compose up`** that brings the whole thing up with one command.
- **A load test** (k6 or similar) that demonstrates your caching/concurrency behavior.
- **Streaming RPC** for large edition lists.
- **OpenAPI** docs for the edge.

Document in your README which stretch goals you attempted and which you deliberately skipped (and
why). "I chose not to do X because Y" is a strong senior signal.

## 3. How we want you to work

### Test-Driven Development

We care about TDD specifically — not just "tests exist." For your **service/business logic**, we'd
like to see tests drive the design: write a failing test, make it pass, refactor. Your commit
history (see §5) is the easiest way to show this. TDD the transport layer loosely if you like;
prioritize the logic.

### Architecture

Aim for a **contract-first, layered, injectable** design. The boundaries that matter to us:

- where does the transport (gRPC / HTTP) end and the business logic begin?
- what is injectable, and what is testable without a running upstream, database, or Redis?
- where does the upstream response get normalized into *your* domain shape, rather than leaking
  upstream field quirks outward?

There is no single right layout. We're evaluating whether your choices are **coherent and
defensible**, not whether they match ours. Be ready to defend the seams.

### Edge cases

Be thoughtful, not exhaustive. We notice: missing `author_name` / `cover_i`, the `-1` cover
sentinel, `description`'s two shapes, empty search results, upstream `429`/`5xx`/timeout, the last
page of pagination, absurd `limit` values, duplicate edition ids, query normalization (case and
whitespace), and a work with hundreds of editions. Handle the ones that matter; note the ones
you'd handle with more time.

## 4. Code review challenge (required)

Below is a real-looking service a teammate wrote. It compiles, and it works fine on a fast
connection against an upstream that never fails — but it has several bugs that will bite in
production. In a **`## Code Review`** section of your README, list every issue you can find,
explain *why* each is a problem, and show how you'd fix it. You don't need to wire this exact type
into your app; we just want your diagnosis.

We're especially interested in *your own* reasoning here. If you use AI on this part, disclose it
per §6 and tell us what you would have missed without it.

```ts
// catalog.service.ts — a teammate's draft
import { Injectable, Logger } from '@nestjs/common';

@Injectable()
export class CatalogService {
  private readonly logger = new Logger(CatalogService.name);
  private readonly cache = new Map<string, any>();

  constructor(private readonly upstream: OpenLibraryClient) {}

  async search(query: string, page: number, limit: number): Promise<any> {
    const key = `search:${query}:${page}:${limit}`;
    const cached = this.cache.get(key);
    if (cached) {
      return cached;
    }

    const result = await this.upstream.search(query, page, limit);
    this.cache.set(key, result);
    return result;
  }

  async getWork(id: string): Promise<any> {
    try {
      const work = await this.upstream.getWork(id);
      const editions = await Promise.all(
        work.editionIds.map((editionId) => this.upstream.getEdition(editionId)),
      );
      return { ...work, editions };
    } catch (error) {
      this.logger.error(error);
    }
  }

  async searchAll(query: string, pages: number): Promise<any[]> {
    const docs: any[] = [];
    for (let page = 1; page <= pages; page++) {
      const result = await this.search(query, page, 100);
      docs.push(...result.docs);
    }
    return docs;
  }
}
```

## 5. Deliverables

Submit a **Git repository** (zip or a private repo link) containing:

1. The project, building and running on a current **Node LTS**. State the Node version in the
   README.
2. The **`.proto` file(s)** and the generated stubs, committed.
3. **Real commit history** — commit as you go. We'll read it to understand your process (and your
   TDD flow). One giant "final commit" is a missed signal.
4. A **README.md** (~1 page) covering:
   - How to build and run the service and the edge, how to run the tests, and a couple of `curl`
     examples.
   - Your architecture and the key decisions/trade-offs behind it (page vs. cursor, cache
     strategy, resilience choices, edge language choice).
   - What you finished, what you skipped, and what you'd do next with more time.
   - Any assumptions you made.
   - A **`## Code Review`** section with your answer to §4.
   - An **`## AI Usage`** section per §6.

## 6. ⚠️ Required: AI usage disclosure

Using AI tools (Copilot, Claude, ChatGPT, Cursor, etc.) is **allowed and not penalized** — we use
them too. But we need to know exactly where, so we can evaluate _your_ reasoning vs. generated
code.

**Two things are required:**

1. **Inline comments** — mark every AI-generated or AI-assisted block with a comment, e.g.:
   ```ts
   // AI-ASSISTED: generated the retry/backoff wrapper, reviewed and edited by me
   ```
2. **A README section** titled `## AI Usage` summarizing which tools you used and for what (e.g.
   "Claude for the gRPC bootstrap and one test scaffold; everything else hand-written").

Undisclosed AI use that we can detect counts against you far more than honest disclosure ever
would. We're hiring your judgment, not a prompt.

## 7. What we're evaluating

| Area                        | What good looks like                                                                                             |
| --------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| **API / contract design**   | A clean `.proto`, a sensible envelope, correct error codes, and an edge that faithfully maps HTTP ↔ gRPC.        |
| **Architecture**            | Clear layer boundaries, dependency injection, a stubbed upstream, a coherent and defensible layout.              |
| **Testing / TDD**           | Tests that drove the design, run offline, and assert real behavior (not just "doesn't crash").                    |
| **Concurrency**             | Bounded fan-out, partial-failure tolerance, deadline/cancellation handling, no data races or leaks.              |
| **Resilience & caching**    | TTL, negative caching, request coalescing, retry with backoff+jitter, timeouts, no stampedes.                    |
| **Error & edge handling**   | Empty vs. not-found vs. upstream-failure vs. invalid-input are distinct; no hanging calls; missing fields okay.  |
| **Diagnosis (§4)**          | Depth of the code review — catching the subtle concurrency/caching/lifecycle issues, not just the obvious ones.  |
| **Code quality**            | Readable, DRY, consistently typed (no lazy `any`), no dead code; commits tell a story.                           |
| **Product judgment**        | Sensible prioritization on the core; clear reasoning about what was cut and why.                                 |

## 8. Ground rules

- **The core is what matters.** If you only have limited time, do the core well rather than rushing
  stretch goals. If you stop before finishing something, write up what you'd do next — that
  write-up is graded.
- Third-party dependencies are allowed but must be justified in the README. Prefer well-understood
  tools over a framework per concern, and prefer the standard library for core logic so we can see
  _your_ work.
- **Tests must run without real network access.** A test that hits Open Library in CI is not a
  test, it's a flaky job.
- You do **not** need any private infrastructure. Keep the scope to one service, one edge, and
  (optionally) one datastore and cache. We're evaluating design and correctness, not scale.
- If anything here is ambiguous, **make a reasonable assumption, document it, and move on.**
  Choosing well under ambiguity is part of the test.

Good luck — we're genuinely looking forward to seeing how you think.
