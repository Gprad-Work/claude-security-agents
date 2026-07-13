# Example: /rule-write → DEDetectionRuleWriter

> Command: `/rule-write "bulk patient record enumeration by a single ClariNote user — sequential ID walking against /api/patients"`
> Agent: `DEDetectionRuleWriter` (Sonnet)
> This scenario comes straight out of the security review — it detects RedTeam **Path 2** (sequential-ID PHI enumeration) and the ThreatIntel handoff for **T1078 / data-access**.

---

## Rule Written

| Field | Value |
|---|---|
| Rule ID | DATA-001 |
| File | `detection-rules/rules/data-access/rule-DATA-001-bulk-patient-enumeration.yml` |
| Title | Bulk patient record enumeration by a single actor |
| Level | high |
| MITRE | T1530, T1078 |
| Efficacy self-check | PASSED — aggregation rule with `timeframe` present; `high` level with 2 FP entries (< 3); no broad `contains` terms; `fields` populated; `condition` includes a filter, not a pure OR |

```yaml
title: Bulk patient record enumeration by a single actor
id: 6f2b1a94-8d3e-4c17-9a2f-1e5c7b0d94af
status: test
description: >
  Detects a single authenticated user reading an unusually high number of distinct
  patient records within a short window — the signature of sequential-ID walking or a
  scripted scrape against /api/patients. Legitimate clinical use touches a small set of
  patients per session; a spike in distinct patient_ids accessed by one actor indicates
  IDOR-driven enumeration or a compromised session harvesting PHI.
author: security-team
date: 2026-07-13
references:
  - https://attack.mitre.org/techniques/T1530/
  - https://attack.mitre.org/techniques/T1078/

logsource:
  category: application
  product: generic

detection:
  selection_patient_read:
    http.route: /api/patients/{patient_id}
    http.method: GET
    response.status: 200

  filter_service_accounts:
    actor.role: service          # batch/reporting jobs legitimately read many records

  condition: selection_patient_read and not filter_service_accounts

  timeframe: 5m
  groupby:
    - actor.user_id
  aggregate:
    distinct_count(patient_id) >= 30

fields:
  - actor.user_id
  - actor.clinic_id
  - src_ip
  - patient_id
  - response.status
  - user_agent

falsepositives:
  - Clinic admin performing a legitimate bulk chart-prep or migration task (verify via change ticket)
  - New-clinic onboarding backfill run under a human account rather than a service account

level: high

tags:
  - attack.collection
  - attack.t1530
  - attack.credential_access
  - attack.t1078
```

---

### Writer's notes (why it's shaped this way)
- **Threshold (30 distinct patients / 5m)** is well above realistic single-clinician session breadth but low enough to catch a for-loop early. Tune to your baseline once you have access telemetry.
- **`distinct_count(patient_id)`** — not raw request count — is the key: it targets *breadth* of access (enumeration) rather than volume (a clinician refreshing one chart).
- **`groupby: actor.user_id`** catches the enumeration even if the attacker rotates source IPs; add `src_ip` to groupby only if you want per-IP granularity.
- **Filter for service accounts** keeps legitimate batch jobs from tripping it — the one common benign bulk-reader — without broadening the detection.
- Pairs with an **auth-anomaly** rule (AUTH category) for the login step of the same attack path; this rule catches the data-access step.

> Next: this rule is handed to `DEReviewRule` — see [`rule-review.md`](rule-review.md).
