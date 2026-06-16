---
name: site-to-api
description: Turn a website into a structured local API. Use this skill when the user provides a URL of a public website and wants to programmatically read structured data from it — for example, "give me an API for this site", "I want to query this page from code", "build me a wrapper for this site", "I need to extract data from here on demand", or "make this site scriptable". The skill guides the agent through a discovery protocol (find the site's private JSON APIs, prerendered state blobs, or HTML structure), then scaffolds a runnable Node.js + Express + Zod server in the current working directory, starts it on a local port, and hands the user back a working API with example requests and responses. Output is a local dev server the user can hit with curl or wire into their own code. Supports common patterns including Next.js sites (look for `__NEXT_DATA__` or `__PRERENDERED_STATE__`), sites with public/private JSON APIs behind their frontend, prerendered SPA shells, and HTML-only sites that need cheerio parsing.
---

# site-to-api

A protocol for turning a public website into a structured local API. Given a URL, the agent investigates the site, identifies the most reliable way to extract structured data, and produces a runnable Node.js server with Zod-validated responses, ready to hit with curl or wire into a larger system.

**Core principle:** Most public websites already have *some* structured data path underneath their rendered HTML — a private JSON API the frontend uses, a prerendered state blob embedded in the page, or a GraphQL endpoint. The protocol is to find that path before falling back to HTML scraping. The result is the same shape in every case: a working server the user can hit locally.

## When to use this skill

**Mandatory when:**

- The user provides a URL of a public website and wants to query it programmatically
- The user says "I need an API for this site", "wrap this site", "make this scriptable", "give me a search endpoint for this", or similar
- The user has a one-off data extraction need from a public site and would otherwise reach for a hosted service like Bright Data, Firecrawl, or Apify
- The user wants to wire a website's data into their own application and is considering scraping it themselves

**Trigger phrases:**

- "Build me an API for [URL]"
- "Give me a local wrapper around [URL]"
- "I need to pull [data] from [site] on demand"
- "Make [site] scriptable"
- "Turn this site into a REST API"
- "How do I get [structured data] out of [site]?"

**If the user wants a one-shot scrape of a single page** and is happy with markdown output, this is overkill — direct them to a hosted scrape skill. This skill is for *reusable* API access to a site's data, with structured responses that fit a programmatic workflow.

**If the user wants a hosted cloud service** (e.g. "deploy this to AWS"), this skill produces a local dev server. Productionizing the output is a separate step.

## The protocol

Six phases. The agent must complete each one in order. Skipping phases produces fragile, hard-to-maintain output.

### Phase 1: Reconnaissance

Before writing any code, study the site. The goal is to map the data surface: what structured data paths does this site expose, and which is the most reliable for the user's intent?

**Step 1.1: Read the homepage**

Use `curl` (or `fetch` in Node) to GET the homepage HTML. Save it to `/tmp/recon/home.html`. Read it. Note:

- What framework is it built with? Look for `<meta name="generator">`, the structure of the HTML, the presence of `__NEXT_DATA__`, `__PRERENDERED_STATE__`, `window.__INITIAL_STATE__`, `__NUXT__`, or similar state blobs
- Are there any obvious JSON endpoints linked from the HTML? Look for `<link rel="alternate">`, `<link rel="preload" as="fetch">`, or `<script>` tags that reference `.json` files
- Are there user-visible forms or search interfaces? Those are clues about the site's data model
- What's the response size? A 50KB HTML page is likely static or prerendered. A 2KB shell with state blob is an SPA

**Step 1.2: Watch the network**

Open the site in a browser (or use Playwright headless) and watch the network panel as the user interacts. Specifically:

- Search for something. Watch what URL gets called
- Click a product / article / detail page. Watch what URL gets called
- Filter or paginate. Watch what URL pattern emerges

The goal is to find the *private JSON API* the frontend uses. Most production sites have one, even if it's not documented. The frontend can't run faster than the network — so if the page renders quickly, there's a JSON endpoint somewhere.

Save the interesting endpoints and their request/response shapes to `/tmp/recon/`. These become the routes of your output API.

**Step 1.3: Check the obvious public APIs**

Some sites have a documented public API that you can use directly. Check:

- The site's `robots.txt` and `sitemap.xml` for canonical URLs
- The footer for "API", "Developers", "RSS" links
- Common patterns: `api.{domain}`, `{domain}/api`, `{domain}/graphql`, `{domain}/feed.json`, `{domain}/rss`

**Step 1.4: Determine the data sources**

By the end of Phase 1, you should have a clear picture of which of these patterns applies:

| Pattern | What to look for | Best extraction strategy |
|---|---|---|
| **A. Public JSON API** | `{domain}/api/v1/...` returns clean JSON | Hit the API directly. Build routes that wrap it. |
| **B. Private JSON API** | The frontend calls `{domain}/api/...` with auth or referer headers; the docs page doesn't mention it | Replicate the frontend's call. The auth surface is usually just a `Referer` header + a session cookie from a prior GET. |
| **C. Prerendered state** | HTML contains `__NEXT_DATA__`, `__PRERENDERED_STATE__`, `window.__INITIAL_STATE__`, or similar JSON blobs | Parse the blob out of the HTML. No network calls needed beyond the initial GET. |
| **D. Server-side rendered HTML** | The page renders the data directly in HTML (classic server-side templating) | Use cheerio to parse. Fall back to this when no JSON path exists. |
| **E. Client-side rendered SPA** | HTML is a thin shell; data fetches happen after JS runs | Use Playwright to wait for the data to load, then read it. Last resort. |

**Multiple patterns may apply.** A Next.js site often has both `__NEXT_DATA__` (pattern C) and a private API the client uses for navigation (pattern B). Use C for the initial page load, B for subsequent data fetches. Document which pattern each route uses.

**Do not skip Phase 1.** The temptation to jump to HTML parsing is strong, but pattern A/B/C are all faster, more reliable, and more durable than pattern D. Ten minutes of reconnaissance saves hours of fragile selector maintenance.

### Phase 2: Schema inference

For each data source you identified, capture one real response and infer the Zod schema.

**Step 2.1: Capture real responses**

Make 3-5 real requests against each endpoint, varying the inputs to see how the response changes. Save them to `/tmp/recon/{endpoint-name}-sample-{n}.json`. Examples:

- For a search endpoint: query with different terms, different filters, different sort orders
- For a detail endpoint: hit 3-5 different product/article IDs
- For a list endpoint: paginate through, filter by category, sort each way

**Step 2.2: Infer the schema**

Look at the captured responses and write a Zod schema that describes the *invariant shape* — the fields that are always present, and the fields that are sometimes present (use `.optional()` or `.nullable()`). Look for:

- Fields that are sometimes `null` vs sometimes a value — use `.nullable()`
- Fields that are sometimes missing entirely — use `.optional()`
- Fields with type variation across responses — use `z.union([...])` or normalize before validation
- Enum-like fields (e.g. `condition: "new" | "used" | "damaged"`) — use `z.enum([...])`
- Nested objects that may be missing — make the parent field optional

**Step 2.3: Validate the schema against real data**

Run every captured response through `Schema.safeParse(response)`. If any fails, adjust the schema until all pass. The schema is your contract — the captured responses are your ground truth.

**Step 2.4: Document the schema's drift behavior**

Note in code comments which fields OLX-like services call "deal signals" — fields that are sometimes present and indicate something specific. Examples: `price.previousValue` (set when a seller dropped the price), `isPromoted` (set when an ad is a paid placement), `validTo` (set on listings with an explicit expiry). These are useful for the user to know about, not just for the agent to handle.

### Phase 3: Scaffolding

Now write the actual server. The output is a Node.js + Express + Zod project in the current working directory.

**Step 3.1: Project layout**

```
./  (current working directory)
├── package.json
├── tsconfig.json
├── .env.example
├── README.md
├── src/
│   ├── index.ts          # Express app entry
│   ├── schemas.ts        # All Zod schemas in one file
│   ├── scrapers/         # One file per data source
│   │   ├── api.ts        # Pattern A/B
│   │   ├── prerendered.ts # Pattern C
│   │   ├── html.ts       # Pattern D
│   │   └── browser.ts    # Pattern E
│   ├── routes/           # One file per route group
│   └── utils/
│       ├── fetcher.ts    # HTTP client with resilience
│       ├── errors.ts     # Custom error types
│       └── cache.ts      # In-memory cache
└── test/
    └── (optional fixtures)
```

**Step 3.2: The fetcher**

Build a single `fetcher` module that handles all outbound HTTP for the target site. It should support:

- A custom `User-Agent` (the user can set it; default to a recent Chrome UA)
- Custom headers per call
- A shared cookie jar (some sites require session cookies from a prior GET)
- Rate limiting (e.g. 1 request per 1-2 seconds, configurable)
- Retry with exponential backoff on 429 and 5xx
- Optional Playwright fallback when the response indicates a challenge (DataDome, Cloudflare, generic captcha)
- Logging: every outbound request should be logged with method, URL, status, duration

This is the resilience layer. A scraper without it will break in production.

**Step 3.3: The error types**

Define error types that map to HTTP status codes. Common types:

- `UpstreamBlockedError` → 503, `retryable: true` (the target site blocked the request; back off and retry)
- `UpstreamParseError` → 502, `retryable: false` (the response didn't match the schema; retrying won't help, the site changed)
- `NotFoundError` → 404 (the resource doesn't exist; don't retry)
- `RateLimitedError` → 429, `retryable: true`

The `retryable: true/false` flag is critical — it tells the user (and the agent calling the API) whether retrying makes sense. Without it, every error looks the same and the caller has to guess.

**Step 3.4: The routes**

One Express router per route group. Each route:

- Validates inputs with a Zod schema
- Calls the appropriate scraper module
- Validates outputs with the response Zod schema (defends against schema drift)
- Maps errors to HTTP responses with the right status code
- Returns JSON

Example route shape:

```ts
router.get('/search/:query', async (req, res) => {
  const query = SearchQuerySchema.safeParse(req.query);
  if (!query.success) {
    return res.status(400).json({ error: 'Invalid query', details: query.error.issues });
  }
  try {
    const results = await searchListings(req.params.query, query.data);
    res.json(results);
  } catch (err) {
    sendError(res, err, 'search');
  }
});
```

**Step 3.5: The drift alarm**

The single most useful feature: a runtime check that validates parsed data against the Zod schema and logs loudly when it fails. Pattern:

```ts
function warnOnDrift(label: string, schema: ZodSchema, value: unknown) {
  const result = schema.safeParse(value);
  if (!result.success) {
    console.error(`[schema-drift] ${label} no longer matches expected shape:`, result.error.issues.slice(0, 3));
  }
}
```

Call this after every parse. When the target site changes its response shape, the operator sees `[schema-drift]` in the logs and knows exactly which endpoint broke and what changed. This is the difference between "the API silently returns garbage" and "the API fails loudly with a specific error."

**Do not throw on drift.** Return the data anyway, with the warning logged. Partial data beats no data for research and prototyping use cases. Production users can opt into strict mode via an env var if they need it.

### Phase 4: Run and test

**Step 4.1: Start the server**

`npm install` then `npm run dev`. The server starts on `localhost:3000` (or whatever port). Confirm the process is alive and listening.

**Step 4.2: Hit every route with curl**

For each route you built, make a real request with a real-looking input. Capture the response. Check:

- Status code is what you expect (200 for hits, 400 for bad input, 404 for not-found)
- Response body is valid JSON
- Response body matches the Zod schema (use `Schema.safeParse` in a one-liner test)
- Response time is reasonable (under 5s for most queries)

**Step 4.3: Test the error paths**

- Send a request with intentionally bad input. Confirm 400 with details.
- Send a request for a non-existent resource. Confirm 404.
- If the site has rate limits, hammer the API until you get a 429. Confirm the response.
- If the site has anti-bot, simulate a blocked request. Confirm the Playwright fallback kicks in or the 503 response is correct.

**Step 4.4: Document the smoke tests**

Put the curl commands and their expected responses in a `## Smoke test` section in the README. The user should be able to copy-paste them and verify the server is working on their machine.

### Phase 5: Hand off

The user now has a working local API. Tell them:

- The server URL (e.g. `http://localhost:3000`)
- The list of routes, with example curl commands for each
- The known limitations (rate limits, missing fields, endpoints you couldn't reverse-engineer)
- How to extend the API (where to add a new route, how to update the schema, how to handle drift)

This is the moment the user can decide if the output is useful. If they say "great, now I want to deploy this," that's a separate conversation about hosting, auth, and persistence — outside this skill's scope.

### Phase 6: Document

The README should contain:

1. **One-paragraph summary** of what the API does and what site it wraps
2. **Quick start** — install, run, hit one endpoint
3. **Endpoint reference** — every route, with method, path, query parameters, example response
4. **Architecture notes** — which patterns (A/B/C/D/E) each route uses, so future maintainers know where to look
5. **Known limits** — rate limits observed, fields the user might want but the site doesn't expose
6. **Disclaimer** — one paragraph. The site owns its data; this API is for the user's own use; respect the site's terms of service and rate limits

The README is half the deliverable. A working server with no README is half a product. A working server with a clear README is the whole thing.

## Site-pattern reference

When Phase 1 reconnaissance identifies a pattern, here's how to handle each:

### Pattern A: Public JSON API

The simplest case. Build routes that wrap the public endpoints. No scraping needed. If the API requires auth (API key, OAuth), document it in the README and add the key to `.env.example`.

### Pattern B: Private JSON API

The site has a JSON API the frontend uses, but it's not documented. To use it:

1. **Capture the exact request the frontend makes.** Use Playwright or a browser dev tools recording. Note the URL, method, headers, cookies.
2. **Reproduce it in your fetcher.** Set the same `User-Agent`, `Referer`, and `Accept-Language`. The site often returns 403 if the request doesn't look like it came from the frontend.
3. **Maintain a session cookie.** Many private APIs require a session cookie from a prior GET to the homepage. Your fetcher should GET the homepage first, then make the API call.
4. **Handle the 403/429 cycle.** Private APIs are more aggressively rate-limited than public ones. Plan for retries with backoff and a polite base delay (1-2s between requests).

### Pattern C: Prerendered state

The HTML response contains a JSON blob with all the data the page rendered. The data is in the HTML — no second request needed.

**For Next.js sites:** look for `<script id="__NEXT_DATA__" type="application/json">`. Parse the JSON inside. The data is keyed by route — you can usually find your specific page's data by walking the object.

**For Nuxt sites:** look for `window.__NUXT__` (a JS variable, not in a script tag). The data is harder to extract because it includes functions and Vue internals — you may need to evaluate it or extract only the JSON-safe parts.

**For generic SSR sites with embedded state:** look for `__INITIAL_STATE__`, `__PRELOADED_STATE__`, `window.__APOLLO_STATE__`, or any other large JSON blob. The naming convention is site-specific; a quick search for `JSON.parse` in the HTML often finds it.

**Build a generic `parsePrerendered` helper** that looks for these patterns in order and returns whatever it finds. This makes the scraper resilient to renames.

### Pattern D: Server-side rendered HTML

The data is in the HTML as text. Use cheerio to parse it. The selectors are fragile — expect to update them when the site redesigns. This is the worst case, but it's also the most common for older sites and small sites.

### Pattern E: Client-side rendered SPA

The HTML is a shell; data fetches happen in the browser after JS runs. Use Playwright to:

1. Open the URL with `waitUntil: 'domcontentloaded'`
2. Wait for a specific selector that indicates the data has loaded (e.g. `await page.waitForSelector('.product-list .item')`)
3. Read the DOM with cheerio after the wait
4. As a faster alternative, intercept the network requests and capture the JSON response directly

Pattern E is the slowest. If the site also has pattern C in the initial HTML, prefer C and use E only for client-side interactions (clicks, filters, infinite scroll).

## Resilience patterns

A scraper without resilience breaks in production. The minimum bar:

- **Rate limiting.** 1 request per 1-2 seconds. Configurable via env var. A user can override it if they know the site allows more.
- **Retry on 429 and 5xx.** Exponential backoff with jitter. Max 2-3 retries. If the third retry fails, return 503.
- **Cookie persistence.** The fetcher maintains a cookie jar across requests in a session. Some sites require this.
- **Schema validation on output.** Every parse validates against the Zod schema. Failures are logged but don't throw — the data is still returned with a `[schema-drift]` warning.
- **Polite User-Agent.** Include a real User-Agent string and ideally a contact URL or email, so the site can reach you if your scraper is misbehaving. Many sites whitelist scrapers that identify themselves.
- **Playwright fallback.** When the regular HTTP client gets a 403/429 with a challenge page in the body, fall back to headless Chromium. This handles most anti-bot measures without needing a third-party service.

## Output the user should see

When this skill is done, the user has:

1. **A working directory** with the new project (`package.json`, `src/`, `README.md`)
2. **A running server** at `http://localhost:3000` (or the configured port)
3. **A README** with the endpoint reference, example curl commands, and known limits
4. **A smoke-test command list** they can run to verify the server is working
5. **A clear list of what the API does NOT do** — so they know its limits

The user can now:
- Hit the API from their own code (`curl`, `requests`, `fetch`, etc.)
- Add their own routes by editing the scrapers and routes directories
- Deploy it to a server if they want it persistent (their problem, not the skill's)
- Stop the server with Ctrl-C when they're done

## Anti-patterns to avoid

**Don't** wrap the entire target site in one `/scrape` endpoint that returns the raw HTML. That's a hosted-scrape service, not an API. The user wants structured data, not HTML.

**Don't** use puppeteer/playwright for everything. They are slow and resource-intensive. Use them only when pattern E is the only option.

**Don't** silently swallow errors. Every error path must produce a structured response with status code, error type, and a hint about whether retrying makes sense.

**Don't** commit the user's data. The server should not log request bodies or response payloads that may contain user-targeted queries. Log URLs, status codes, and durations. That's enough for debugging.

**Don't** skip the README. A working server with no documentation is half a product. The README is part of the deliverable.

**Don't** assume the user wants MCP. MCP is one way to expose the API; curl is another; library bindings are a third. Default to a plain HTTP API. Mention MCP as a follow-up the user can opt into.

**Don't** try to handle every site generically. Each site has its own quirks. The protocol above is the *approach*, not a copy-paste template. Adapt it to the site you're actually wrapping.

## When the user pushes back

If the user says "just do it faster" or "skip the reconnaissance, just scrape the HTML": push back. The reconnaissance is what separates a scraper that works in week 1 and breaks in week 3 from one that works for months. The cost of 10 minutes of reconnaissance is much less than the cost of rewriting selectors every time the site redesigns.

If the user says "I don't need the README" or "skip the docs": push back. The README is how the user remembers what they built. Without it, the server is a black box they have to reverse-engineer every time they want to extend it.

If the user says "I need this to handle 10,000 requests per second": this is the wrong skill. Direct them to a hosted scraping service (Bright Data, Apify, Firecrawl) or a proper API gateway design. The output of this skill is a local dev server, not a production system.

## What this skill is not

- **Not a hosted scraping service.** It produces a local dev server, not a cloud product.
- **Not a generic web scraper.** It's specifically for the "wrap a site as a structured API" workflow, not for one-off page reads.
- **Not a no-code tool.** It produces a real TypeScript project. The user has to run `npm install` and have Node.js installed.
- **Not a replacement for the site's own API.** If the site has a public API, the user should use that directly. This skill is for sites where no public API exists.
- **Not a copyright workaround.** The data on the site is the site's. The user is responsible for respecting copyright, terms of service, robots.txt, and rate limits. The README's disclaimer should make this clear.
- **Not an MCP server by default.** The output is a REST API. The user can opt into an MCP wrapper as a follow-up step, but it's not the default.
- **Not a one-shot scraper.** The user can call `curl https://...` and get a single response, but the value of this skill is the *reusable* server the user can call repeatedly.

## When the user just wants markdown

If the user has a one-off need for "give me the content of this page as text," they don't need a server at all. Direct them to a hosted scrape skill or a simple `curl | pandoc -f html -t markdown` pipeline. This skill is overkill for that use case.
