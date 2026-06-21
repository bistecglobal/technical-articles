# AI-Generated Load Tests: Turning an API Contract into a Running k6 Script

*How BistecGlobal built a Claude Code skill that converts an OpenAPI spec, URL list, or plain-English API description into a ready-to-run k6 performance test script — no k6 expertise required.*

Performance testing is one of those engineering disciplines most teams agree they should do — and most teams skip when deadlines hit. The barrier isn't ignorance; developers know they need load tests. The barrier is the tax of writing them: knowing k6's API, translating endpoint semantics into realistic user flows, picking the right load profile, and wiring in thresholds that actually mean something.

At BistecGlobal, we shipped a Claude Code skill — `k6-test-generator` — that removes that tax. You hand it an OpenAPI spec, a list of URL patterns, or a plain-English description of your API. It hands back a ready-to-run k6 script, a test data file, exact run commands, and guidance on reading the results. No k6 expertise required to start.

## The Problem: Performance Testing as a Bottleneck

The `k6-test-generator` skill was designed around a documented problem: QA engineers, developers, and DevOps engineers need performance tests but lack the k6 depth to write them efficiently from scratch. Writing a good k6 script is surprisingly involved.

You need to know the right import structure, how to define `options.stages` for ramp-up/hold/ramp-down, how to parameterize test data using `SharedArray` to avoid memory overhead, how to write `check()` assertions that fail the test when thresholds are breached, and how to handle dynamic values like auth tokens extracted from earlier responses. That's meaningful up-front work before you've even run your first test.

The result, as the skill's design goal addresses: teams without k6 expertise either wrote ad-hoc scripts that missed k6 best practices, or skipped performance testing altogether.

## The Solution: Three Input Modes, One Usable Script

The skill accepts three input modes, reflecting the actual states developers find themselves in when they need performance tests:

**1. OpenAPI/Swagger spec** — the generator parses all paths and methods, groups endpoints by resource (e.g., all `/users/*` operations together), generates sample request bodies from schema definitions, creates a logical CRUD flow where the API structure suggests one, and includes authentication setup if `securityDefinitions` are present. Deprecated endpoints are skipped unless explicitly requested.

**2. URL patterns** — a list of endpoints with HTTP methods and optional sample payloads. The generator identifies path parameters (`:id`, `{id}`, `{userId}`), infers reasonable request bodies for POST/PUT operations, and organizes endpoints into logical user flows where a sequence is implied.

**3. Plain-English description** — describe what your API does in natural language. The generator infers the test structure, identifies the key user journeys, and builds a script around them.

Regardless of input mode, the output is always the same four parts: the complete k6 script, a test data file (CSV or JSON where parameterization makes sense), the exact `k6 run` command with recommended flags, and a brief guide on what to look for in the results.

## Design Decisions Worth Calling Out

### Standard Load Profiles as First-Class Citizens

The skill formalizes five standard load profiles and presents them explicitly to users rather than burying the choice in documentation:

| Profile | Pattern | Use Case |
|---------|---------|----------|
| Smoke   | 1–5 VUs, 1 min | Verify the script runs at all |
| Load    | Ramp up, hold, ramp down | Baseline load capacity |
| Stress  | Ramp beyond normal load | Find breaking points |
| Spike   | Sudden burst | Flash sale / viral event simulation |
| Soak    | Steady moderate load, long duration | Surface memory leaks and degradation |

This matters because most "performance tests" in the wild are really just load tests — they check whether the system survives expected traffic. Stress, spike, and soak tests catch different failure modes, and making those profiles explicit encourages teams to think about which scenario they're actually trying to simulate.

### Baked-In k6 Best Practices

Rather than generating naive scripts that pass basic execution, the skill encodes k6 best practices as non-negotiable generation rules:

- **`SharedArray` for test data** — avoids re-parsing the same JSON on every VU iteration, which matters at scale
- **`check()` on every request** — status code assertions plus optional response body validation
- **`group()` blocks** — organises related requests so results are readable in k6's output and cloud dashboards
- **`sleep(1–3)` between requests** — simulates realistic think time; without it, VUs hammer endpoints unrealistically
- **Correlation** — extracts dynamic values (IDs, tokens) from responses and threads them into subsequent requests
- **`__ENV.BASE_URL`** — keeps the target host out of the script so the same test runs against local, staging, and production without edits

These aren't hints; they're rules the generator applies automatically. A team inherits years of k6 field knowledge on their first generated test.

### Sensible Default Thresholds

The skill ships defaults that represent a reasonable starting point for most REST APIs:

```javascript
thresholds: {
  http_req_duration: ['p(95)<500', 'p(99)<1500'],
  http_req_failed: ['rate<0.01'],
  http_reqs: ['rate>100'],
}
```

p95 under 500ms, p99 under 1500ms, error rate below 1%, throughput above 100 requests/second. These thresholds gate the test — if the system doesn't meet them, the run exits with a non-zero code, which fails a CI pipeline. Teams can tighten or loosen thresholds in the generated script; the defaults give them a starting baseline grounded in common SLO targets.

## Where This Fits in the ai-core Testing Skill Suite

`k6-test-generator` was shipped alongside two related skills that form a tiered QA capability:

- `bistec-qa` — the foundational testing standards skill, covering xUnit patterns, Playwright E2E, k6 integration, four-tier quality gates, and a Definition of Done framework
- `perf-tests` — a deeper hybrid performance testing skill that combines Playwright browser recording with k6 script generation, with 24 workflow files covering create/edit/validate modes

`k6-test-generator` sits between these two: more opinionated than `bistec-qa`'s guidance, more accessible than `perf-tests`'s full workflow. It's the right entry point for a developer who needs a working script today and doesn't yet have a performance testing infrastructure.

## Using It in Practice

A typical invocation looks like:

> "Generate k6 tests from this OpenAPI spec: [paste swagger.json]. Use 30 virtual users and set a threshold of p95 < 300ms."

Or for a team without a spec:

> "I have a REST API for an OKR platform. Main flows are: list objectives, create a key result, update progress, fetch team dashboard. Generate k6 tests simulating realistic usage with 50 concurrent users."

The skill also handles the authentication case that trips up most from-scratch k6 scripts: if your API requires a Bearer token, you describe the login endpoint and the generator wires `setup()` → login → extract token → thread token into all subsequent requests via `data`.

## The Outcome

The primary outcome is measured in adoption friction, not benchmark numbers: a developer or QA engineer who has never written a k6 test can go from zero to an executable, production-ready test script in a single conversation. The generated script covers the practices that would otherwise require reading k6's documentation, studying parameterization patterns, and understanding why `SharedArray` beats a plain array — all of which add up to the hour or two that causes performance tests to slide into the backlog.

The skill is available across all BistecGlobal projects that install the ai-core plugin. Teams can invoke it against any API surface — internal microservices, third-party integrations, or customer-facing endpoints — without any additional setup.

## Lessons Learned

Three things shaped the final design:

**Input flexibility matters more than output perfection.** Teams reach for performance test tooling at different stages: some have a full OpenAPI spec, some have a URL list from Postman, some have neither and just know what their API does. Supporting all three input modes ensures the tool is useful at the moment teams actually need it, not just when they have ideal inputs.

**Default thresholds should be opinionated but visible.** Generating a test with no thresholds produces a script that always passes regardless of performance. Generating thresholds without surfacing them produces false confidence. The design settles on sensible defaults that are explicit in the output, so teams can immediately see what pass/fail means and adjust accordingly.

**Encoding best practices in generation rules beats documenting them separately.** The `SharedArray` pattern, correlation, environment variables — these live as generation rules in the skill definition, not as a separate best-practices document. The result is that a generated script is its own teaching artifact: developers reading it encounter the patterns in context, not in the abstract.

---

*Originally published at bistecglobal.com*

**BistecGlobal** builds AI-native enterprise software. Follow us for engineering insights.
