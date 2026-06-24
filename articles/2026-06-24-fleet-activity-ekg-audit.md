# Audit — 2026-06-24-fleet-activity-ekg

**Verdict: PASS**
**Score: 24/25**

## Claim inventory & verdicts

| # | Claim | Verdict | Evidence |
|---|-------|---------|----------|
| 1 | MCD runs a fleet of Claude agents, one subprocess per project | ✅ | Established architecture |
| 2 | Five event sources: alerts, injects, memory, digests, broadcasts | ✅ | `EkgSourceKey` union; `SOURCE_ORDER` |
| 3 | `getEkgTimestamps`: alerts/injects split by `alert_type`; broadcasts ISO→epoch via `strftime('%s')`; `deleted_at IS NULL` | ✅ | `db.ts:652-666` verbatim |
| 4 | 48-hour window, hourly bins (`WINDOW_HOURS=48`, `HOUR=3600`) | ✅ | `api/ekg/route.ts:5-6` |
| 5 | `now` snapped to top of hour; `startHour`; idx = floor div; out-of-range guard | ✅ | `route.ts:40-63` verbatim |
| 6 | Bins pre-built zero-filled, oldest→newest | ✅ | `Array.from({length: WINDOW_HOURS})` with zeroed counts |
| 7 | Per-lane independent scaling, `Math.max(1, …)` floor | ✅ | `ekg/page.tsx:41-45` |
| 8 | Bar height `n===0?2:Math.max(3, round((n/max)*40))` | ✅ | `page.tsx:88-108` verbatim |
| 9 | Source colours: alerts red, injects cyan, memory purple, digests amber, broadcasts green | ✅ | `SOURCE_COLOR` map `page.tsx:9-15` |
| 10 | Hover surfaces hour + exact count | ✅ | bar `title={hourLabel · label: n}` + `onMouseEnter` |
| 11 | Busiest hour across all sources combined, surfaced as headline | ✅ | `route.ts:73-78` busiestHour walk on `b.total` |
| 12 | Refreshes every 60s off dedicated `/api/ekg` | ✅ | `useFreshness('/api/ekg', 60_000)` |
| 13 | Reading interpretations (flatline, cross-lane spike, diurnal rhythm) | ✅ | Valid reading of the multi-lane waveform; presented as interpretation |

No fabricated metrics; "2pm yesterday"/"twice a day" are explicitly illustrative.

## Forward-looking scan
None (0 matches).

## SHAs / PR numbers in prose
None.

## Rubric

| Dimension | Score | Notes |
|---|---|---|
| Evidence quality | 5 | Every claim maps to verbatim source |
| Technical depth | 5 | Multi-table timestamp pull, hour-snap, per-lane scaling, ISO reconciliation all accurate |
| Clarity for audience | 5 | EKG/heartbeat framing strong and sustained |
| BistecGlobal voice | 5 | Practitioner-focused, pull-only-what-you-plot lesson |
| Title specificity | 4 | Specific + human; "Five Event Streams on One Waveform" earns it |

**Total: 24/25 — PASS**
