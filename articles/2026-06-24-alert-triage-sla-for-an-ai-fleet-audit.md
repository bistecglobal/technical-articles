# Audit — Your AI Fleet's Alerts Are Lying to You

**Verdict: PASS** · **Score: 24/25** · Claims fixed: 0 · Claims flagged: 0

## Claim inventory

| # | Claim | Verdict | Evidence |
|---|-------|---------|----------|
| 1 | Alerting started append-only via `insertAlertEvent` | ✅ | `src/db.ts` `insertAlertEvent(slug, alertType, description, payload)` |
| 2 | Migration adds nullable `ack_ts` + `ack_by`, idempotent at startup | ✅ | commit `9a88fd4` db.ts `ALTER TABLE alert_events ADD COLUMN ...` in try/catch |
| 3 | `NULL` ack_ts is the open state | ✅ | `acknowledgeAlert` predicate `ack_ts IS NULL`; `includeAcked===false` → `ack_ts IS NULL` |
| 4 | Guarded UPDATE, returns `changes > 0`, prevents double-ack | ✅ | `acknowledgeAlert` body, `WHERE id=? AND ack_ts IS NULL` |
| 5 | `unacknowledgeAlert` inverts the transition | ✅ | db.ts `SET ack_ts=NULL, ack_by='' WHERE id=? AND ack_ts IS NOT NULL` |
| 6 | `getAlertEvents` gained `includeAcked` default true (no regression) | ✅ | db.ts signature + `includeAcked` default comment |
| 7 | Alert calendar + alert-flow flip to exclude acked | ✅ | commit `9a88fd4` message + `getAlertFlow(sinceTs, includeAcked=true)` ackClause |
| 8 | Ack endpoint session-gated, 401 if no session | ✅ | `app/api/alerts/[id]/ack/route.ts` auth check |
| 9 | POST acks, DELETE re-opens; both write `alert.ack`/`alert.unack` audit | ✅ | ack route both handlers `insertAuditLog({verb:'alert.ack'...})` |
| 10 | Validates integer id + existence (400/404) | ✅ | `Number.isInteger` → 400; `!getAlertEvent` → 404 |
| 11 | `ack_by` actor = logged actor | ✅ | same `actor` var passed to both `acknowledgeAlert` and `insertAuditLog` |
| 12 | SLA route reads raw 3 cols over 30-day window | ✅ | `getAlertSlaRows` SELECT alert_type, ts, ack_ts; `DAYS_BACK=30` |
| 13 | Percentiles/ack-rate/backlog computed in handler, single pass | ✅ | `alert-sla/route.ts` GET loop + map |
| 14 | Latency = `ack_ts - ts`; open feeds backlog + oldest-open age | ✅ | route handler if/else branches |
| 15 | Percentiles use linear interpolation between ranks | ✅ | `percentile()` `lo + (hi-lo)*(idx-lo)` |
| 16 | Sorted worst-first: open backlog then slower median | ✅ | `types.sort((a,b)=> b.openBacklog-a.openBacklog || (b.medianSec??0)-(a.medianSec??0))` |

All 16 claims ✅ verified against merged code on `main` (commits `9a88fd4` P196, `e6fa35e` P198).

## Forward-looking scan
No matches for will / plan to / coming soon / roadmap / next step / soon. Outcomes framed historically. PASS.

## Rubric

| Dimension | Score |
|---|---|
| Evidence quality (no raw SHAs in prose) | 5 |
| Technical depth | 5 |
| Clarity for audience | 5 |
| BistecGlobal voice | 5 |
| Title specificity | 4 |

**Total: 24/25** — exceeds 20 threshold. No fixes required.
