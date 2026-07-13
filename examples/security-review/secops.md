# Example: SecOps on ClariNote PRD

> Agent: `SecOps` (Sonnet) · Input: [`../sample-system/PRD.md`](../sample-system/PRD.md)
> Illustrative domain output.

---

## SecOps Review

### Critical findings
| # | Capability | Finding | Blind spot / scenario | Fix |
|---|---|---|---|---|
| S-001 | Detection — PHI access anomalies | No alerting on PHI access volume or failed-authorization spikes (§6). Given sequential patient IDs and thin authz (see AppSec A-001), enumeration is the top risk — and it would be completely silent. | An attacker walks `/api/patients/{1..N}`. There is no detection on read volume per user, per patient breadth, or 403/404 rate. The first signal is a breach notification. | Alert on: per-user patient-access breadth over a rolling window, spikes in 403/404 on `/api/patients`, and bulk summary reads. Baseline per-role normal and page on deviation. |

### High findings
| # | Capability | Finding | Blind spot / scenario | Fix |
|---|---|---|---|---|
| S-002 | Audit logging — PHI access | No immutable audit trail of who accessed which patient/summary (§6). Logs go to CloudWatch but PHI-access events are not called out, and there's no tamper-resistance. | After an incident, responders cannot reconstruct which patients an attacker viewed — a HIPAA breach-scoping failure (ties to GRC G-004). | Emit a structured, append-only PHI-access event (actor, clinic, patient, action, result, ts) to a WORM/retained store separate from app logs. |
| S-003 | Logging hygiene — PHI in logs | On error, the full request body is logged to CloudWatch, and Sentry captures request context (§6). Both now contain PHI in lower-controlled pipelines. | A logging pipeline compromise or over-broad log access exposes PHI outside the database's controls; also a detection *source* becomes a *liability*. | Redact PHI/bodies before logging; scrub Sentry request context. (Detailed under DataSecurity D-001 — SecOps flags the detection/response impact.) |
| S-004 | Forensic readiness — CloudTrail scope | CloudTrail is enabled in the primary region only (§6). Activity in other regions (including attacker-created resources) is unlogged. | An attacker operating cross-region, or exfil via another region, leaves no trail. Incident reconstruction is partial. | Enable CloudTrail org-wide, all regions, with log-file validation and a locked-down log bucket. |

### Medium / Low findings
| # | Capability | Finding | Blind spot / scenario | Fix |
|---|---|---|---|---|
| S-005 | IR readiness | No documented incident-response plan or on-call/escalation path (§6, §9). | When an alert does fire, there's no runbook, severity model, or containment plan. | Write an IR plan with severities, on-call, and PHI-breach playbook; test it. |
| S-006 | Egress / exfil detection | No egress monitoring on workers, which hold elevated S3/vector access. | Bulk data movement to an external destination would be invisible (ties to RedTeam exfil paths). | Add egress anomaly detection / DLP on worker traffic; alert on unusual destinations and volumes. |

### What's done well
- Centralized logging (CloudWatch) and error tracking (Sentry) exist — the pipeline is present; it needs PHI redaction and detection content layered on, not net-new infrastructure.

### Verdict
**HIGH RISK** — The system can generate logs but cannot *detect* the most likely attack (silent PHI enumeration, S-001) or *reconstruct* it afterward (S-002, S-004). For a PHI system, detection and forensic readiness are not optional. Prioritize PHI-access alerting and immutable audit logging before production.
