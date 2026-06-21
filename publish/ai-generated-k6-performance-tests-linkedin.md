Writing a k6 performance test from scratch takes more expertise than most teams have time to build — so we made an AI skill that writes it for you.

At BistecGlobal, we shipped `k6-test-generator` as a Claude Code skill inside our ai-core plugin. Give it an OpenAPI spec, a URL list, or a plain-English description of your API — it returns a complete, ready-to-run k6 script.

Key design choices:

• **3 input modes** — OpenAPI/Swagger spec, URL patterns, or plain English. Meets teams wherever they are, not just when they have ideal inputs.

• **5 standard load profiles built in** — Smoke, Load, Stress, Spike, and Soak. Most teams only run load tests; making the others explicit pushes teams to think about which failure mode they're actually testing for.

• **Best practices enforced, not suggested** — SharedArray for test data, check() on every request, sleep() for think time, __ENV.BASE_URL for portability. These are generation rules, not documentation.

• **Opinionated defaults** — p95 < 500ms, p99 < 1500ms, error rate < 1%. Thresholds gate the test: miss them and CI fails. Visible by design so teams know exactly what pass/fail means.

The outcome: a developer who has never written a k6 test can go from zero to an executable, production-ready script in a single conversation.

Full article → [link placeholder — operator fills in after Medium publish]

#PerformanceTesting #k6 #DeveloperTools #AI #DevOps
